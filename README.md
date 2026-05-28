# ECO Paper Presentation

A paper-club presentation deck (and speaker notes) for:

> **ECO: An LLM-Driven Efficient Code Optimizer for Warehouse-Scale Computers**
> Hannah Lin et al., Google — arXiv:2503.15669 (March 19, 2025)

The deck explains how ECO uses an LLM to automatically find and fix performance
anti-patterns across Google's production C++ code, framed for an audience that
studies LLM-driven "evolve / discovery" systems but is newer to this specific
domain. Examples are translated to Python where possible.

## Quick start

Open the deck in any modern browser — no build step, no server:

```
eco_deck.html
```

It's a self-contained [reveal.js](https://revealjs.com/) presentation that loads
the library from a CDN, so an internet connection is needed the first time.

### Presenter controls

| Key | Action |
|---|---|
| `→` / `Space` | Next slide |
| `←` | Previous slide |
| `F` | Fullscreen |
| `S` | Speaker / presenter view |
| `Esc` | Slide overview grid |

## Repo contents

| Path | What it is |
|---|---|
| `eco_deck.html` | The reveal.js presentation (13 slides). Self-contained. |
| `eco_speaker_notes.md` | Plain-language narration for every slide — read this end-to-end before presenting. |
| `images/` | Chart images and diagrams used by the deck (figures from the paper + the Moore's Law plot). |
| `paper/2503.15669v1.pdf` | The source paper. |

## Slide structure

| # | Slide |
|---|---|
| 1 | Title |
| 2 | Why this paper exists (Moore's Law, scale) |
| 3 | ECO in one line (headline numbers) |
| 4 | End-to-end example — one commit, start to finish (visual hook) |
| 5 | Anti-pattern catalog — 3 of 7, with Python examples |
| 6 | Stage 1 — Learn: build the recipe library (offline / one-time) |
| 7 | The online loop — Locate → Generate → Verify (interactive) |
| 8 | Stage 2 deep dive — Performance Opportunity Localization (interactive) |
| 9 | Stage 3 deep dive — Code Edit Generation Strategies |
| 10 | Stage 4 deep dive — Verification, Submission & Monitoring |
| 11 | Results — a year in production (interactive) |
| 12 | Open question — the evolve → discovery analogue |
| 13 | Q&A |

## Interactive features

A few slides go beyond static content. **These respond to mouse clicks — don't
use arrow keys while interacting, since those advance the deck.**

- **Slide 7 (online pipeline):** click each stage box (Locate → Generate →
  Verify) to swap the detail panel below. Resets to Locate when you re-enter
  the slide.
- **Slide 8 (Stage 2 deep dive):** click each step box (Parse + profile →
  Embed & retrieve → Re-rank) to reveal a jargon panel underneath.
- **Slide 11 (results):** click each chart to show its explanation below.
- **Clickable jargon popups:** several technical terms across slides 2, 5, 8,
  and 9 open a popup with a deeper, plain-language explanation when clicked
  (they have a dotted underline and a small "(click for…)" hint). Examples:
  *code* (Google's monorepo), *Move* / *Vector* (anti-pattern categories),
  *AST*, *GWP*, *Embedding + ScaNN*, *MAP@5*, and the prompting-strategy
  evaluation. Click the **× close** button, click the dimmed backdrop, or
  change slides to dismiss.

## Notes

- The deck is tuned for projector readability: large fonts, sparse slides.
- Anti-pattern code examples use Python; the paper itself is C++. The patterns
  are universal, but ECO's specific recipe catalog is C++-shaped.
