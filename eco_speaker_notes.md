# ECO — Speaker Notes

> Talking script for the ECO paper review presentation.
> Read end-to-end at least once before the talk. Each section maps 1:1 to a slide in `eco_deck.html`.
> The **Say** lines are written in spoken language — feel free to adapt the phrasing, but the *content* should land.

---

## Slide 1 — Title

**On screen:** Paper title, authors (Google), year.

**Say:**
> Hi everyone. Today's paper is from Google, published in 2025. It's called ECO — short for Efficient Code Optimizer. The headline is that they built a system that uses an LLM to automatically improve the performance of code running in Google's production data centers. Over the past year of running it, it's saved them roughly half a million CPU cores worth of compute every quarter.

**Transition:**
> Before I dive into how it works, let me explain why anyone would build something like this in the first place.

---

## Slide 2 — Why this paper exists

**On screen:** Bullets on Moore's Law, Google scale, hand-tuning bottleneck, today's tools — alongside a Moore's Law transistor-count chart (1970–2020).

**Say:**
> For decades, software just got faster because hardware got faster. That's Moore's Law — you can see the chart on the right: transistor count per chip has been doubling every couple of years for fifty years, and it's still going up. But here's the catch the chart *doesn't* show: while transistor count kept doubling, the *per-core performance* gains stopped doubling around the mid-2000s. The line is still going up, but the performance we used to get for free from it has flattened out. So we can't count on hardware to bail us out anymore.
>
> Meanwhile, at Google's scale, even tiny efficiency wins matter. If you shave one percent off the CPU usage of a function that runs on every web request, that one percent times billions of requests adds up to the same compute as a small data center. Real money, real electricity.
>
> But here's the catch: writing those optimizations is slow, manual work. You need an engineer who understands the code, profiles it, finds the hot spot, writes the fix, gets it reviewed, ships it, watches for regressions. Google has billions of lines of code and not enough engineers to do this for every line.
>
> Today's tools — compilers, and static analyzers like Clang-Tidy — catch some easy wins. Quick note on what those tools are, since they're C++-flavored: **Clang-Tidy is basically the C++ analogue of `pylint` or `ruff` in our world.** It's a linter that scans code against a fixed catalog of rules — things like "this variable should be marked `const`," "this shadows another variable," "this loop has a known inefficient shape." Compilers do something similar at compile time, applying a fixed set of optimizations they know are always safe.
>
> Both are great at *syntactic* patterns — things you can spot by looking at the shape of the code without really understanding what it does. But they can't catch optimizations that require *understanding the intent* of the code — and that's where most of the interesting wins live. That gap is exactly what ECO is trying to fill.

**Jargon decoded:**
- *Moore's Law:* a rough rule that compute power doubles every couple of years. Has slowed down a lot.
- *Clang-Tidy:* C++ static-analysis tool — think `pylint` / `ruff` for Python. Scans code against a fixed rule catalog. Catches syntactic problems, not semantic ones.
- *Static analyzer / linter:* a tool that reads your code *without running it* and flags issues by pattern-matching. Fast and safe, but limited.

**Transition:**
> So the question is: can an LLM do that "understanding" part — at scale, automatically, and reliably enough to actually ship the fixes? That's what ECO is.

---

## Slide 3 — ECO in one line

**On screen:** One-line definition + three big numbers + framing line.

**Say:**
> Here's ECO in one sentence: it's a system that uses a large language model to *automatically* rewrite slow code in Google's production fleet, and *actually ship* the changes — not just propose them.
>
> Three numbers to anchor what they accomplished. Over a year of running in production, ECO landed more than six thousand four hundred commits. More than ninety-nine and a half percent of them worked correctly — almost no regressions. And the resulting compute savings are roughly half a million CPU cores per quarter, every quarter.
>
> The reason I picked this paper for us: it sits in the same family as the "evolve" papers we've been studying — LLMs proposing code edits, a system evaluating which ones are good. So instead of spending half the talk re-explaining that paradigm, I want to focus on what makes ECO *different* from a typical evolve paper.

**Transition:**
> Before I dig into the details of how it works, let me plant a picture in your head — one real ECO commit, end to end.

---

## Slide 4 — ECO in action: end-to-end example (visual hook)

**On screen:** Two-column layout. **Left:** the end-to-end diagram from the paper showing one concrete ECO commit. **Right:** four numbered steps in plain English explaining the flow.

**Say:**
> Before I dig into how ECO works, I want to plant a picture in your head. The image on the left is straight from the paper — one real ECO commit, traced end to end. On the right, the same flow in four plain-language steps.
>
> **Step one — find the slow spot.** Google's fleet profiler is constantly measuring CPU usage across all production code. ECO uses that to find functions that are both *expensive* AND that *look like* one of the known anti-patterns we'll see in a minute.
>
> **Step two — write the fix.** A fine-tuned LLM proposes a code edit, using the matching anti-pattern recipe as a template. The LLM has already been trained on tens of thousands of similar fixes from Google's history, so it's not starting from scratch.
>
> **Step three — check it's safe.** This is the part most LLM-codegen research skips. Three layers: run the file's existing tests, have the LLM review its own work against a checklist, then send to the actual human code-owner for final sign-off.
>
> **Step four — ship and watch.** Merge to production. The fleet profiler then keeps watching that function. If performance actually got worse, or behavior is weird, ECO auto-reverts the change.
>
> Don't try to memorize the steps now. The point is: **ECO is a multi-stage system, not a one-shot LLM call.** Refer back to this image as we drill into each step in the rest of the talk.

**Transition:**
> So now let me show you the kinds of patterns ECO actually fixes.

---

## Slide 5 — Anti-pattern catalog (3 of 7 in detail)

**On screen:** Two-column layout. **Left:** three anti-patterns stacked vertically, each with a side-by-side Python before/after. **Right:** the bar chart showing all 7 anti-pattern categories ECO mined from commit history — with a caption explaining what each axis means.

**Say:**
> Quick orientation on the chart on the right — it's straight from the paper. The **x-axis** is the 7 anti-pattern categories ECO discovered when mining Google's commit history. The **y-axis** is the number of historical commits ECO found for each category, on a **log scale** — that's important: each gridline is ten times the one below it.
>
> Seven categories total: Alloc, Args, Copy, Map, Move, Sort, and Vector. *Sort* is the biggest by far — almost a hundred thousand commits — but the paper focuses its detailed evaluation on three: **Copy, Map, and Vector**, which is what I have on the left.
>
> Why those three and not Sort? The paper doesn't say it in one sentence, but the practical reason is: Sort's commits are mostly comparator and stability fixes, which are correctness work rather than pure performance. Copy, Map, and Vector are locally scoped, semantically simple, and easy to verify — the sweet spot for ECO's verification pipeline.
>
> The paper's examples are C++; I've translated to Python. Quickly:
>
> **#1 — Unnecessary copy.** Making a copy of something when you only need to read it. The "before" version allocates a million-element copy and immediately throws it away; the "after" just iterates the original.
>
> **#2 — Redundant lookup.** Looking up the same key three times where once would do. Each hash lookup is cheap, but multiplied by a hot loop, it adds up.
>
> **#3 — Growing a collection.** Building a list with `.append()` in a loop, which re-allocates as the list grows. The list-comprehension version skips most of that.

**Transition:**
> So those are three examples of *what* ECO does. The rest of the talk is about *how*. Let me show you the system in one interactive view.

---

## Slide 6 — The 4-stage pipeline (interactive — all on one slide)

**On screen:** A pipeline diagram at the top with four clickable stage boxes (1. Learn → 2. Locate → 3. Generate → 4. Verify). Below it, a detail panel that shows the content of whichever stage is active. **Default on entry: Stage 1 highlighted, its detail visible.**

**How to drive it:** Click each stage box in order. The active stage gets a glowing yellow border; the others dim. Don't use arrow keys mid-slide (they advance the deck) — click only. Leaving and returning to the slide resets to Stage 1.

**Say (overview, before clicking anything):**
> This is the whole system on one slide. Four stages, in order — Learn, Locate, Generate, Verify — connected as a loop because the production telemetry from the last stage feeds back as a signal for the next round. I'll click into each box and we'll spend a couple of minutes on what's inside.

### Click Stage 1 — Learn

**Say:**
> Stage one — Learn. The whole point is: *don't invent optimizations, mine them.* Google has decades of commit history, and many commits are explicitly tagged as performance improvements — either in the commit message, by winning an internal speed award, or by having attached benchmark deltas. ECO scans roughly fifty-five thousand of these. For each one, it pulls out the *before* code and the *after* code. Then it clusters the similar ones together using unsupervised methods. That gives the seven anti-pattern categories we just saw: Alloc, Args, Copy, Map, Move, Sort, and Vector.
>
> Each cluster becomes a "recipe" — a known problem plus concrete examples of how engineers actually fix it. So at the end of Stage 1, ECO has a library of seven recipes, each grounded in real human work.

### Click Stage 2 — Locate

**Say:**
> Stage two — Locate. This is the most interesting stage, in my opinion, because it's what makes the whole system *cheap enough* to run.
>
> The question is: given the seven recipes, *where in billions of lines of code should we apply them?* They use two signals together.
>
> First signal: the fleet profiler. Google runs a continuously-on CPU profiler across all production code — it's already there, collecting telemetry. ECO reuses that to filter all functions down to roughly *ten million costly ones*. The rest aren't worth touching.
>
> Second signal: embeddings. Each function gets turned into a vector of numbers — a compact representation that captures its structure. The seven anti-patterns get the same treatment. Then for each anti-pattern, find the ~500 nearest function-vectors. Code-aware embeddings work about three times better than text-based ones — which makes sense, because two functions can look similar character-by-character and behave totally differently.
>
> Then re-rank those 500 with stricter code-similarity metrics — control flow, types, structure — to get a high-confidence shortlist.
>
> The result: the LLM is never called blind on a random function. It only ever sees candidates the system already has good reason to think are fixable. That's the "strong prior" — and it's the first big differentiator we'll come back to.

### Click Stage 3 — Generate

**Say:**
> Stage three — Generate. This is the part that maps directly onto the evolve papers we already know.
>
> They take Gemini Pro 1.0 and fine-tune it on those fifty-five thousand historical commits. So the model has *seen* what good performance edits look like in Google's own codebase before it ever writes a new one.
>
> They try four prompting strategies. Zero-shot: just describe the task and show the code. Few-shot: include two or three before/after examples in the prompt. Chain-of-thought: ask the model to reason step-by-step before producing the edit. ReAct: let the model take actions — read other files, call simple tools, iterate.
>
> The finding: no single strategy wins everywhere. Zero-shot and ReAct lead overall, but specific anti-patterns favor specific strategies. So ECO doesn't pick one; it keeps a *library* and matches strategy to anti-pattern. This is the part you'll recognize from prior papers — the engineering is in the matching, not the LLM itself.

### Click Stage 4 — Verify

**Say:**
> Stage four — Verify. This is what makes ECO actually shippable, and it's where most evolve research stops.
>
> Four layers, in order. *Layer one*: run the existing CI tests on the edited code. If the build breaks, ECO tries to auto-fix simple stuff — missing includes, small syntax fixups. If tests fail, the edit gets dropped.
>
> *Layer two*: LLM self-review. The same model — or a checker model — is shown the edit and asked, against a checklist: "Did you change the semantics? Could this introduce undefined behavior? Did you accidentally remove a side effect?" Catches a meaningful chunk of subtle bugs that pass tests but would still be wrong.
>
> *Layer three*: human code-owner review. The edit goes through Google's normal code review process — the human who owns the file reviews it, accepts or rejects. No bypass. ECO is treated like any other contributor.
>
> *Layer four*: post-deploy monitoring. After the commit lands and runs in production, the profiler keeps watching. If performance actually regressed, or behavior is weird, auto-revert.
>
> Net result: production success rate over ninety-nine and a half percent, revert rate under half a percent. This is what makes the system real instead of a research prototype — and it's our second differentiator.

**Transition (after all four):**
> Now that you've seen the four stages, let's pull back and ask the question I promised: what makes ECO *different* from a typical evolve paper? Three things — that's the next few slides.

---

## Slide 7 — Stage 1 zoomed in: how the recipe library is built

**On screen:** Two-column layout. **Left:** bullets on the sources ECO mines and what comes out the other end. **Right:** the data-mining pipeline diagram from the paper (Figure 4) showing the flow from raw sources → filter/group → named anti-pattern categories.

**Say:**
> Quick zoom-in on Stage 1, because there's more to it than I let on in the interactive view. ECO doesn't only mine commits — it pulls from a few different sources and combines them.
>
> Source one is the obvious one: historical commits flagged by keyword search. Words like *"time/op," "speed up," "speeds up,"* or specific phrases like *"replace std::string& with absl::string_view"* or *"avoid copies."*
>
> Source two: design docs and analysis documents. Inside Google, when someone writes an internal doc describing a performance optimization, that doc usually references the commits that did it. So ECO follows those references in reverse — find the doc, find the commits it points to.
>
> Source three: curated sources. Performance tips and tutorials engineers have written. Prior large-scale refactors that touched many files. Commits flagged by static-analysis tools like Clang-Tidy.
>
> All those candidate commits then go through a filter step: remove anything that got reverted, anything that wasn't actually a performance fix, anything where the before-and-after doesn't tell a clean story. Then group what's left into named categories — you can see some of them on the right: `explicit_move`, `string_view`, `suboptimal_map_or_set`, `vector_reserve`, and so on. There's also an "other/unclassified" bucket — the catch-all.
>
> Net result: a Performance Anti-Pattern Data Set — the recipe library that the rest of ECO depends on. Worth noting for our team: this is **bootstrapped from the company's own engineering history**. A Python team would have to do this exercise for itself, and the categories would look completely different.

**Transition:**
> Next stage: how does ECO find places in the codebase where those recipes apply?

---

## Slide 8 — Stage 2 zoomed in: Performance Opportunity Localization (interactive)

**On screen:** Three numbered step boxes in a horizontal flow with arrows — "Parse + profile" → "Embed & retrieve" → "Re-rank" — each clickable, each with a short bullet list. A jargon-explanation panel below updates when you click a step. **Default on entry:** Step 1 active.

**How to drive it:** Click each step box. Active step glows yellow, others dim. The detail panel underneath shows jargon definitions for the active step. Don't use arrow keys mid-slide.

**Say (overview, before clicking):**
> Same zoom-in pattern, now for Stage 2. This is the most engineering-dense stage of ECO and arguably the one that sets it apart from a typical evolve paper. Paper calls this section "Performance Opportunity Localization." Three sub-steps. Each one has its own bit of jargon — I'll click each step to surface the definitions as we go.

### Click Step 1 — Parse + profile

**Say:**
> Step one is about narrowing the codebase down to where it's worth looking.
>
> Take every C++ function and parse it with Clang to get an **AST** — Abstract Syntax Tree — basically a structured tree representation of the code's grammar, stripped of variable names and whitespace. That lets us compare functions by their *shape* rather than their text.
>
> Then attach CPU usage data from Google's **fleet profiler** — that's their always-on, fleet-wide CPU measurement system. The data is already being collected; ECO just reuses it. Filter down to about ten million "costly" functions — everything else isn't worth bothering with.
>
> One clever detail: **re-attribution.** Generic library functions like `std::vector::push_back` get a ton of CPU time on the profiler, but you can't really optimize them in isolation — they're called from a million places. Re-attribution pushes that CPU cost back to the *application code* that called the library. So ECO ends up working on functions where edits are actually meaningful.

### Click Step 2 — Embed & retrieve

**Say:**
> Step two takes those ten million costly functions and matches them against known anti-patterns.
>
> Turn each function into an **embedding** — a vector of numbers. Similar code gets nearby vectors. Same trick that powers semantic search. Do the same for each anti-pattern example in the recipe library.
>
> Then use **ScaNN** — Google's open-source nearest-neighbor search library — to find, for each anti-pattern, the top ~500 functions whose embeddings are closest. ScaNN is the magic that makes this fast enough at scale; comparing all pairs by brute force would be hopeless.
>
> The paper tested three embedding methods. Bag-of-words just counts tokens — fast and dumb. Deep text embeddings use a general-purpose text model. Deep code embeddings use a model trained specifically on code structure — and it wins by about three times on **MAP@5** (Mean Average Precision at 5; basically: out of the top 5 retrieved results, how many were actually relevant). Code-aware understanding matters.

### Click Step 3 — Re-rank

**Say:**
> Step three cleans up the top 500 with stricter criteria.
>
> They build a composite similarity score across four signals. **BLEU** and **ROUGE-L** are text-similarity metrics borrowed from machine translation — BLEU looks at how much the n-grams (token sequences) of two pieces of code overlap; ROUGE-L looks at the longest common subsequence, which is more sensitive to ordering. They're used here to compare code structurally.
>
> **Type-set intersection** asks: do the two functions use the same C++ types? Both use `std::string` and `std::vector<int>`? That's a strong signal they're doing similar things.
>
> **Control-flow similarity** asks: do the loops, branches, and returns line up between the two functions? Focuses on the *shape* of the logic.
>
> Before computing any of this, they strip variable names and comments — because the structural shape is what matters, not whether someone named a variable `foo` or `bar`.
>
> The result is a much smaller, much more confident shortlist per anti-pattern.

**Transition (after all three):**
> Bottom line: out of billions of lines of code, you end up with a hand-pickable list of candidates for each anti-pattern. The LLM in Stage 3 never gets called blind — only on functions that are hot AND look like a known anti-pattern. That's the *strong prior* that makes ECO work.

---

## Slide 9 — Stage 3 zoomed in: Code Edit Generation Strategies

**On screen:** Four cards in a row, one per prompting strategy (Zero-shot, Few-shot, Chain-of-Thought, ReAct), each with a short description and a "vibe" note. Below: a highlighted callout with the key finding — no single winner; ECO picks per recipe.

**Say:**
> Same zoom-in pattern, now for Stage 3 — the LLM actually writing the edit. The paper's section heading is "Code Edit Generation Strategies."
>
> Setup is straightforward. Take Gemini Pro 1.0 — the production Google LLM at the time — and **fine-tune** it on those fifty-five thousand historical performance commits we talked about. So the model has *already seen* what good performance edits look like in Google's own codebase before it ever writes a new one.
>
> Then the interesting question: *how do you prompt this fine-tuned model?* They tested four strategies — shown on the slide.
>
> **Zero-shot** — the cheapest. Just describe the task and show the code, ask for the fix. No examples, no reasoning, no tools. It's the strong baseline.
>
> **Few-shot** — include two or three before/after examples in the prompt. Useful when the pattern is subtle and the model needs to see exactly what kind of edit you want.
>
> **Chain-of-Thought** — ask the model to reason step-by-step before producing the edit. This gives *more diverse* edits — the model explores different fixes — but also has a higher rate of invalid output, because more exploration means more wrong turns.
>
> **ReAct** — the most powerful. The model can take actions: read other files with `cat`, apply patches, iterate. This is best when context matters — when the right fix depends on what's happening elsewhere in the codebase.
>
> Key finding from the paper: **no single strategy wins everywhere**. Zero-shot and ReAct lead overall on the similarity-to-human-fix metric. But specific anti-patterns prefer specific strategies. Copy edits do well with zero-shot. Vector edits often need ReAct. So ECO doesn't pick one; it maintains a *library* of strategies and matches each anti-pattern recipe to the strategy that works best for it.
>
> This is the part of the system that maps most directly onto the evolve papers you already know — same shape, more careful per-recipe tuning.

**Jargon decoded:**
- *Zero-shot / Few-shot:* prompting techniques. "Zero-shot" = no examples in the prompt. "Few-shot" = a handful of examples shown to the model as templates.
- *Chain-of-Thought (CoT):* a prompting trick where you tell the model to "think step by step" before answering. Often improves reasoning quality, at the cost of more variability.
- *ReAct:* "Reasoning + Acting" — a prompting pattern where the model alternates between thinking and taking actions (like calling tools). Lets the model gather information mid-task.
- *Fine-tuning:* continuing to train the base model on a domain-specific dataset, so it gets better at that domain.

**Transition:**
> Once the LLM has written an edit, the most important question is: *do we trust it enough to ship to production?* That's Stage 4.

---

## Slide 10 — Stage 4 zoomed in: Verification, Submission & Monitoring

**On screen:** Four numbered layers in a horizontal flow with arrows — CI tests → LLM self-review → Human review → Post-deploy monitor — each with bullets and a "catches" line at the bottom. Below: a callout with the headline numbers (>99.5% success, <0.5% reverts).

**Say:**
> Stage 4 is the gauntlet. This is where most evolve research stops — they show a benchmark passes and call it a day. ECO can't stop there because the goal is *production deployment*. Four layers, each catching a different kind of failure.
>
> **Layer one — CI tests.** Run the file's existing unit and integration tests on the edited code. If the build breaks, ECO tries to auto-fix simple stuff: missing includes, small syntax errors. If tests still fail, the edit is dropped. *Catches functional regressions* — the kind of breakage where the code doesn't do what it used to.
>
> **Layer two — LLM self-review.** The model is shown its own edit and asked, against a checklist of common mistakes: *"Did you change the function's semantics? Could this introduce undefined behavior? Did you accidentally remove a side effect?"* *Catches subtle semantic bugs that slip past tests* — the kind where tests pass but the code is still wrong for some edge case.
>
> **Layer three — human code-owner review.** The edit is submitted as a normal code change request. The human who owns that file reviews it — same process as any human-authored PR. They can reject it for any reason. **Never skipped.** *Catches design and intent issues* — "this looks correct, but it's not the direction I want this code going."
>
> **Layer four — post-deploy monitoring.** After the commit lands and runs in production, the profiler keeps watching. If the change actually made the function *slower*, or caused weird behavior — auto-revert. *Catches production-only failures* — concurrency bugs, real-workload issues, things you can't see in pre-deploy testing.
>
> Net result: *over ninety-nine and a half percent success rate* in production, *less than half a percent* of merged commits get reverted.
>
> The engineering insight worth landing: **four imperfect filters that catch different categories of failure** beat one strong filter. Tests catch functional bugs but miss semantics. Self-review catches semantics but misses design. Humans catch design but can be tired or wrong. Monitoring catches everything else. Stack them and you get a system reliable enough to actually ship.

**Transition:**
> So those are all four stages. You may recognize the shape from the end-to-end image we opened with — each box in that picture corresponds to one of these four stages. Now let's see what they all add up to in production.

---

## Slide 11 — Results: a year in production (interactive)

**On screen:** Stats row at the top (5 headline numbers). Three clickable chart cards below. A detail panel beneath the cards that updates when you click one. **Default on entry:** chart 1 selected, its explanation showing.

**How to drive it:** Click each chart card in order (or jump around). Active chart glows yellow; others dim. Same caveat as the pipeline slide — don't use arrow keys mid-slide, they advance the deck. Leaving and returning resets to chart 1.

**Say (overview, before clicking anything):**
> One slide for the headline results. The five numbers at the top tell the high-level story.
>
> Over the past year, ECO landed about **six thousand four hundred commits**. That's roughly **twenty-five thousand lines of code** touched. Production success rate **over ninety-nine and a half percent** — almost no regressions. Compute savings come out to about **half a million normalized CPU cores per quarter**, which is **over two million normalized cores total** across the year.
>
> Quick reminder on what *normalized cores* means: Google operates many different machine types, with different CPU speeds. Normalized cores — NC for short — is their way of converting saved compute into "equivalent number of standard CPU cores you don't need anymore." So this is real, fungible compute.
>
> Three charts below. I'll click into each one and we'll spend a minute on what it's telling us.

### Click Chart 1 — Edits per anti-pattern (Fig. 14a)

**Say:**
> First chart — edits per anti-pattern. Each bar group is one of the seven anti-pattern categories from earlier. The **blue bars** are the number of ECO commits submitted for that category. The **orange bars** are the lines of code touched.
>
> Notice the volume distribution: *Copy* and *Vector* dominate, around ten thousand commits each. *Map* is much lower, around a few hundred. Why? Because Copy and Vector fixes are simple and locally scoped — adding a `const&` or a `vector.reserve()` is a small, predictable edit. Map fixes are more complex; removing an unnecessary map lookup often requires understanding more of the surrounding code. So fewer Map edits get through the verification gauntlet, but each one is more involved.

### Click Chart 2 — Compute savings per anti-pattern (Fig. 14b)

**Say:**
> Second chart — compute savings per anti-pattern. Same x-axis, but now the y-axis is normalized cores saved on a log scale.
>
> Here's the interesting twist: the volume chart from before *doesn't* tell you the savings story. *Vector* edits in particular are short but high-impact — pre-allocating one container can avoid a whole stream of downstream allocations across the rest of the program. *Map* has fewer commits but still meaningful savings, because each successful Map edit tends to be on a hot path.
>
> The lesson: **volume of edits is not the same as compute saved**. You have to look at both charts together to understand the trade-off ECO is making.

### Click Chart 3 — Landed edits over time (Fig. 15)

**Say:**
> Third chart — edits over time, across the full year. The **blue** line is total lines of code touched, cumulative. The **yellow** line is lines modified, **green** is added, **red** is deleted.
>
> Two things to notice. One: modifications dominate. Most ECO edits are *tweaks to existing code*, not wholesale rewrites or large new chunks. This is consistent with the anti-pattern catalog we saw — these are surgical fixes.
>
> Two: the trend is upward with no plateau. ECO isn't a one-off experiment that produced its wins and stopped. It's a continuously-running system that's been quietly accumulating wins for a year, and the slope suggests there's room to keep growing as the anti-pattern catalog expands and LLM capabilities improve.

**Transition (after all three):**
> So those are the numbers, in three charts. Now let me leave you with the open question I think is most worth our team's attention.

---

## Slide 12 — Open question for the room

**On screen:** Two big lines posing the open question about evolve → discovery.

**Say:**
> Here's the question I want to throw to the room, especially the seniors.
>
> ECO works because Google has something extraordinary: a fleet profiler that gives them a continuous, ground-truth signal about how their code performs in reality. Without it, ECO would just be another LLM-rewrites-code paper.
>
> We're shifting from evolve toward discovery. Evolve has clear fitness signals — benchmarks, unit tests, "this-program-runs-faster." Discovery is harder. What's the "production telemetry" for discovering new scientific results, or new algorithms, or new proofs?
>
> Maybe it's experimental validation against real-world data. Maybe it's running against a curated benchmark like a math olympiad set. Maybe we need to build the equivalent of a fleet profiler for the domain we care about.
>
> I don't have an answer. But ECO's success comes specifically from having a *real* fitness signal, not a synthetic one. As we move toward discovery, the most important question might not be "what model do we use" — it might be "what's our fitness signal, and is it real enough?"
>
> I'd like to hear your thoughts.

**Transition:**
> Let me wrap up with the three things worth remembering.

---

## Slide 13 — Q&A

**On screen:** "Q&A" title + three takeaway bullets + paper citation.

**Say:**
> To wrap up — three things worth remembering from ECO, beyond the headline numbers.
>
> First: evolution works much better when you have a *strong prior* about where to mutate. ECO doesn't blast the LLM at random functions; it uses profiler data plus similarity search to pre-pick. That's Stage 2.
>
> Second: production deployment needs a *layered verification loop*, not a single test. Four imperfect filters that catch different things beat one strong filter. That's Stage 4.
>
> Third: ECO's fitness signal is real production telemetry, not a benchmark. That's the part most distinct from typical evolve papers.
>
> Paper is on arXiv: 2503.15669. Happy to take questions.

**Anticipated questions:**
- *"Couldn't a compiler do this?"* Compilers do some of it, but they have to be conservative (must not change semantics). ECO can be more aggressive because human review is in the loop.
- *"What about other languages?"* Paper covers C++ only. The system is in principle language-agnostic, but you'd need to retrain the model and rebuild the recipe library per language.
- *"Why didn't they use a stronger LLM?"* Gemini Pro 1.0 was current at the time of work. The bottleneck isn't model size — it's the verification pipeline and the localization.
- *"What's the cost?"* Paper doesn't give a clean dollar figure, but they note the cost of LLM calls is small compared to the ongoing CPU savings.
- *"Is this code/system released?"* No. Internal to Google.

---

## End-of-deck notes for the presenter

- **Interactive slides (pipeline + results):** Click items to advance the bottom panel. Both slides reset to item 1 when re-entered, so don't navigate away mid-explanation. If you accidentally hit an arrow key, you'll jump to the next slide — press the back arrow to return; the panel resets.
- **If you're running long:** the entire interactive slide can be shortened by skipping Stage 3 (Generate) — it's the most familiar part for an evolve-paper audience and the least distinctive.
- **If a senior pushes back with "but evolve papers also have multi-stage evaluation":** you're right, some do. ECO's distinction is that the *production deploy itself* is the evaluation, not a sandboxed proxy. Concede the point and clarify.
- **If anyone asks about the Python examples being misleading:** yes, they're translations. The paper is C++. The patterns are universal but the specific recipes ECO uses are C++-shaped.
- **If you blank during Q&A:** revisit the stats row on slide 11 (Results). "Two million normalized cores saved" and ">99.5% production success" are the cleanest sound-bites from the paper.
