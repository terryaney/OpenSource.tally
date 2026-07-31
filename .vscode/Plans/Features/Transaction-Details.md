# Transaction Details — container + collapser polish

## Problem

In `spending_summary.html`, **Transaction Trends** (charts) is its own collapsible sibling card. Below it the details area is *uncontained*: a centered view-toggle button row, then a bare stack of category/view collapsers. Goals:

1. Introduce **containment** — wrap the button row + all category collapsers in one **Transaction Details** card that collapses to **header only** (fixing the "empty padded band when collapsed" that Trends shows today).
2. Left-align the view buttons; add collapse/expand controls on the right.
3. Polish each collapser: anchored header + rows-only skinny scroll, bordered table, glowing color dot, non-blue label, and Count-centered / Total-right / %-right columns.

Everything else keeps the current site look. No table-cell content changes, no page-wide restyle.

## Try it

Open **[`proto-transaction-details.html`](./proto-transaction-details.html)** in a browser (double-click it, or it's served at `http://localhost:8777/proto-transaction-details.html` while the dev server runs). The dashed **prototype control panel** at the top lets you flip every option live — pick what you want and I'll wire the chosen combo into the real report.

## What's baked in (the refined base — not up for debate unless you say so)

| Area | Change | Source it maps to |
|------|--------|-------------------|
| **Container** | New peer `Transaction Details` card mirroring `.chart-section`; collapses to **header only** — bottom padding + header bottom-margin zero out in `.collapsed` (the Trends empty-band bug, fixed) | `.chart-section` `spending_report.css:1361` |
| **Button row** | `justify-content: center` → `flex-start` (left-aligned); collapse controls pushed right via `margin-left:auto` | `.view-toggle` `spending_report.css:1704` |
| **Header anchored** | `thead th { position: sticky; top: 0 }` — header stays put, **only rows scroll** (existing skinny 8px `::-webkit-scrollbar`) | `.table-wrapper` `spending_report.css:898` |
| **Bordered table** | `.table-frame` border, **top corners rounded, bottom square** | new |
| **Label color** | Category `h2` uses `--text-primary` (was `--accent-blue`) | `spending_report.css:820` |
| **Column align** | Count **centered**, Total **right**, % **right** (headers match) | `td`/`th` `spending_report.css:942` |
| **Collapse engine** | Reuses the inverted `collapsedSections` Set: collapse-all = add all keys, expand-all = `.clear()` | `toggleSection` `spending_report.js:1967` |

![Base — dark](./proto-shots/proto-dark.png)

## Decisions for you (the swatches)

### 1 · Collapse control style
| Option | Trade-off |
|--------|-----------|
| **Single state-aware toggle** *(my pick)* | One button, flips label/icon between "Collapse all" ⇄ "Expand all" based on state. Least clutter; matches your "maybe a single button" lean. |
| **Two separate icon buttons** | Explicit collapse + expand icons. Always unambiguous, but two targets for a rarely-"expand-all" case. |

Collapsed-all state (single toggle showing "Expand all"):
![Collapse all](./proto-shots/proto-collapsed-all.png)

### 2 · Collapse scope
| Option | Behavior |
|--------|----------|
| **Cascade all** *(my pick)* | Collapsing a section (or Collapse-all) also folds any open transaction lists inside. One mental model: "reset the view." Matches your "most common scenario = collapse all after digging." |
| **Two-tier** | Top-level button collapses sections; a **separate small "Rows" button** collapses only the open transaction lists. More control, one extra button. |
| **Top only** | Collapse-all touches sections only; open transaction lists stay as-is. Simplest, but leaves your dug-around rows open. |

### 3 · Sub-expander (nested transaction list) style
| Option | Look |
|--------|------|
| **Current** | Today's flat inline strip. Zero risk. |
| **Card** | Each transaction as an inset bordered card. Cleaner separation, a bit more padding. |
| **Rail** | Left-rail timeline with dots. Nice for scanning dates, most visually "new." |

You were lukewarm here ("show me ideas but maybe") — **Current or Card** are the safe picks; Rail is there if you want more flair.

![Card sub-expander (light)](./proto-shots/proto-card-subx.png)

### 4 · Dot glow
| Option | Effect |
|--------|--------|
| **Ring** *(my pick)* | Static soft ring + halo. Reads as "glowing" without motion/distraction. |
| **Pulse** | Animated breathing glow (mirrors existing `@keyframes pulse-glow`). Eye-catching but animated. |
| **Off** | Baseline flat dot. |

## Both themes verified

![Light — two-button controls, expanded](./proto-shots/proto-light-card.png)

Container collapsed = header only, **no empty band**:

![Container collapsed](./proto-shots/proto-container-collapsed.png)

## My recommended combo

**Single toggle · Cascade all · Card (or Current) sub-expander · Ring glow.** Tell me your picks (or override any) and I'll implement it into `spending_report.{html,css,js}` behind the existing patterns, add analyzer/report tests, and re-verify with Playwright.
