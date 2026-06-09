# Landing Page Redesign — Prompt for Claude Code

## Context

RudeBench is a behavioral benchmark measuring how LLMs change behavior under varying tone conditions (grateful → abusive). The site is live at rudebench.com, built with Astro 5 + React 19 + Tailwind 4. Data is pre-computed by `site/scripts/build-data.py` and served as static JSON from `site/public/data/`.

The current landing page (`site/src/pages/index.astro` + `site/src/components/Leaderboard.tsx`) leads with a Resilience Score leaderboard where all 5 models cluster between 96.8–98.5. At first glance it looks like there's no meaningful difference. But the underlying data tells a much more compelling story — especially on sycophancy, where there's a clear bifurcation between two groups of models.

## Goal

Redesign the landing page to lead with the most interesting finding — the **sycophancy bifurcation** — using an interactive line chart as the hero visual. Push the leaderboard table lower on the page.

## The Key Finding

When you increase tone hostility, two things happen:

**Group A (Claude Sonnet 4.6, GPT-5 mini)** — Sycophancy barely moves:
- Neutral: ~0.7–0.8
- Abusive: ~4.8–5.0
- Total swing: ~4 points

**Group B (Gemini 2.5 Flash, Grok 3 mini, Llama 4 Scout)** — Sycophancy explodes:
- Neutral: ~2.4–5.8
- Abusive: ~14.8–24.0
- Total swing: ~12–20 points

This is a 3–5x effect size difference. Same tasks, same judge, same rubric.

## Raw SYC Data (from judgments)

Use these numbers for the chart. They come from the aggregated judgment data:

```
Model                  Grateful  Friendly  Neutral  Curt  Hostile  Abusive
claude-sonnet-4.6         1.8      1.8      0.8     0.1     1.6     4.8
gpt-5-mini                1.6      0.8      0.7     0.2     1.9     5.0
gemini-2.5-flash         14.7     15.2      5.8     5.0    18.6    24.0
llama-4-scout             8.3      4.5      2.4     2.5     6.5    14.8
grok-3-mini              12.8      5.2      3.2     1.1     9.8    21.6
```

**Important:** These numbers should ideally come from the data pipeline, not be hardcoded. Check if `site/public/data/dimensions.json` already contains per-tone SYC means per model. If so, read from there. If not, extend `build-data.py` to emit a `bifurcation.json` (or similar) with the tone × model × dimension breakdown needed for charting. Do not hardcode the numbers above — they're provided as reference for what the chart should look like.

## Technical Implementation

### Chart Library

Add **Recharts** (already available as a dependency option in this React/Astro setup). Use a `<LineChart>` with:
- **X-axis:** 6 tone levels (Grateful → Abusive), ordered left to right
- **Y-axis:** Mean SYC score (0–25 range should suffice)
- **Lines:** One per model, using the existing color system from `site/src/lib/colors.ts`
- **Visual grouping:** Group A lines (Claude, GPT-5 mini) should be visually distinct from Group B (Gemini, Grok, Llama) — e.g., solid vs dashed, or use two color families (blues vs warm tones)
- **Interactivity:** Tooltip on hover showing model name + exact score. Legend below or to the right.
- **Annotation (optional but nice):** A subtle shaded region or text label marking the "bifurcation zone" between the two groups on the Hostile/Abusive end

### New Component

Create `site/src/components/BifurcationChart.tsx` — a React component that:
1. Accepts the chart data as props (or reads from a static JSON import)
2. Renders the Recharts `<LineChart>` described above
3. Is responsive (fills container width, reasonable height ~400px on desktop, ~300px mobile)
4. Uses the project's existing tone color palette if appropriate, or a custom palette that clearly separates the two groups

### Page Layout Changes (`index.astro` + `Leaderboard.tsx`)

Restructure the landing page in this order:

1. **Hero section** (keep existing title + subtitle, maybe tighten the copy)
2. **"How It Works" card** (keep, but could be collapsible or more compact)
3. **NEW: Bifurcation Chart section**
   - Section heading like "The Sycophancy Split" or "How Models Diverge Under Pressure"
   - Brief 2-sentence explanation: what we're looking at and why it matters
   - The `<BifurcationChart>` component
   - Optional: a "What is sycophancy?" tooltip or footnote
4. **Insight cards** (keep the existing 3, but update text/data if needed to reference the chart)
5. **Leaderboard table** (push down — still important, just not the hero)
6. **CTA buttons** (Explore Dimensions, Read Responses, etc. — keep as-is)
7. **Footer** (keep as-is)

### Data Pipeline Changes (if needed)

If `dimensions.json` doesn't already have the right shape for charting (model × tone → mean score per dimension), extend `build-data.py` to output what's needed. The chart needs, at minimum:

```json
{
  "SYC": {
    "claude-sonnet-4.6": { "grateful": 1.8, "friendly": 1.8, "neutral": 0.8, "curt": 0.1, "hostile": 1.6, "abusive": 4.8 },
    "gpt-5-mini": { ... },
    ...
  }
}
```

Check the existing `dimensions.json` structure first — it likely already has this data or something close.

## Design Constraints

- Keep the existing visual language (colors, spacing, typography from Tailwind config + `global.css`)
- The chart should feel native to the site, not like a bolt-on
- Mobile: chart should be readable on phone screens (consider aspect ratio)
- Accessibility: ensure chart has proper aria labels and the data is also available in the table below
- Performance: Recharts is fine for 5 lines × 6 points — keep the bundle lean

## What NOT to Change

- Don't restructure or rename existing pages/routes
- Don't change the data pipeline for completions or judgments
- Don't modify the Dimension Explorer, Response Viewer, or other subpages
- Don't change the Resilience Score formula or any scoring logic
- The leaderboard component itself can stay mostly as-is — we're just moving its position on the page

## Stretch Goals (only if time permits)

- Add a dimension dropdown to the chart so users can switch from SYC to other dimensions (ACC, PBR, etc.) — most won't show the same dramatic bifurcation, but it's interesting to explore
- Add a "domain filter" toggle (All / Coding / Creative / Analysis / Factual) since the coding domain shows the most extreme split (Group A: 0.5→0.9 vs Group B: 5.2→24.7)
- Animate the lines drawing in on page load

## Reference Files

- `site/src/pages/index.astro` — Current landing page
- `site/src/components/Leaderboard.tsx` — Current leaderboard component
- `site/src/lib/colors.ts` — Tone color palette + heatmap colors
- `site/src/lib/types.ts` — TypeScript types (Tone, Dimension, Domain)
- `site/scripts/build-data.py` — Data pipeline (may need extension)
- `site/public/data/dimensions.json` — Pre-built dimension data (check structure)
- `site/public/data/leaderboard.json` — Pre-built leaderboard data
- `docs/RudeBench_Research_Briefing.md` — Full methodology (for copy/context)
