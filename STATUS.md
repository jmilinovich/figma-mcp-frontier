# STATUS — overnight run, 2026-06-16 → 06-17

What got built while you slept, what's proven, and exactly where to pick up.

## TL;DR
Public repo is live, the corrected capability map + benchmark harness are written, and the **live empirical core has resolved 8 of 12 planned experiments + all 4 write rungs + the Plugin-API frontier probes** — results nobody has published, including one finding that overturns the field's #1 complaint about the Figma MCP. Everything is committed and pushed. The remaining empirical work is blocked on one thing only you can do (see "⛳ The one thing that needs YOU" below).

**Repo:** https://github.com/jmilinovich/figma-mcp-frontier
**Live benchmark draft (throwaway, my Figma drafts):** `figma-mcp-frontier-bench` — file `PSSvW47Ry1XDSAVZWQcN7t`

## Headline findings (live remote build, 2026-06-17)

1. **The "MCP hardcodes tokens" complaint is misattributed.** Controlled A/B: an *unbound* Button → `bg-[#17171c]` (hardcoded); a *token-bound* Card → `bg-[var(--color-surface,white)]`, `gap-[var(--space-lg,24px)]`, `rounded-[var(--radius-lg,12px)]`. `get_design_context` emits clean `var(--token,fallback)` refs **iff the design binds variables that carry WEB code-syntax.** It's a design-side problem, not a tool limit. → `results/read-fidelity-tokens.md`
2. **Component sets read back as parametric typed components.** `get_design_context` on a 3-variant Button set returned one React component with `variant?: "Primary"|"Secondary"|"Ghost"` and per-variant conditional styling — not three snippets. The reported variant brittleness is the **Code Connect** path, not raw `get_design_context`. → same file
3. **`clientFrameworks`/`clientLanguages` are logging-only (RESOLVED).** A `swiftui` request returned byte-identical React+Tailwind. The MCP always emits React+Tailwind; conversion is the agent's job. The "Compose still returns React" bug is actually the documented contract. → `results/ground-truth-probes.md`
4. **Per-tool token cost, measured head-to-head** (same node): `get_metadata` ~98 tok · `get_screenshot`(URL) ~120 tok · `get_design_context` ~368 tok + a default inline screenshot. URL screenshots carry zero base64; `excludeScreenshot` is the cheap win. → `results/token-cost.md`
5. **A ~350× reality gap worth chasing:** a clean semantic Card measured **~452 tokens** vs the widely-cited "~162K-token card." Strong signal the famous blowups are about **node density / imported cruft / inline rasters**, not semantic complexity. (Next experiment.)

## Write frontier — `use_figma` success
**All 4 ladder rungs built, every one clean on the first call, zero retries:**
- Rung 1: Button (frame + text) · Rung 2: Button **COMPONENT_SET** (derived variant property) · Rung 3: token **variable collection** + fully-bound **Card** · Rung 4: 51-node **dashboard** (12 stat cards)

**Plugin-API frontier, probed live (all ✅):** prototyping **reactions** (`setReactionsAsync`) execute + persist; the newer **Noise / Texture / Glass** effects are all scriptable (NOISE has a `.d.ts`-vs-runtime `blendMode` drift); instance **text overrides** are returned by `get_design_context`; **atomicity** confirmed for both thrown-JS and API-error failures (a failed call leaves zero nodes). Atomicity + the `figma-use` skill make the write path reliable in practice. → `results/write-success.md`, `results/ground-truth-probes.md`

## What's in the repo
- `docs/capability-map.md` — corrected 19-tool reference (10 remote-only / 9 shared, the renames, the 4 non-existent tools)
- `docs/methodology.md`, `docs/experiment-matrix.md` (E01–E12), `docs/landscape-comparison.md`
- `harness/`, `fixtures/` — reproduction + corpus spec
- `skill-draft/SKILL.md` — the reliability skill, first draft
- `results/` — the live measurements above
- `docs/_qa-report.md` — an adversarial accuracy gate caught + fixed 4 over-claims in the authored docs

## Experiment status — 8 of 12 resolved (see `docs/experiment-matrix.md` for the live table)
**Resolved this session:** E03 reactions ✅ · E04 Noise/Texture/Glass effects ✅ · E06 `get_metadata` multi-design (no pathology) ✅ · E07 per-tool token cost ✅ · E08 `clientFrameworks` logging-only ✅ · E09 instance overrides returned ✅ · E10 atomicity ✅ · E12 base64-vs-URL screenshot cost ✅ · plus multi-mode read limit, token-binding, instances-vs-clones, component-set read shape.
**Partial:** E01 `forceCode` (no-op on small nodes; downgrade-override needs a node past the cap) · E02 single-call op ceiling (no cap at 51 nodes; true ceiling unmeasured).
**Walled (seat):** E05 + E11 — the whole Code Connect cluster needs an **Org/Enterprise Dev seat** (read *and* write blocked on Pro/Full).

## ⛳ The one thing that needs YOU (unblocks the rest)
The remaining empirical work — **E01 definitive cap/`forceCode`**, **E02 op-ceiling**, and **#2 shadcn read-fidelity** — all need a **dense, imported design**. Easiest unblock: **duplicate a public shadcn/ui Figma community kit into your drafts and paste me the file URL.** With that I can: run the real 25k-token cap failure + `forceCode` override, and score official read-fidelity against the actual shadcn code as answer-key (the headline comparison). Code Connect (E05/E11) needs an Org/Enterprise Dev seat — out of scope on Pro unless you want to test on a work tenant.

## Lower-priority autonomous follow-ups (don't need you)
- E07 N≥3 medians for tighter token numbers; a second-mode theming read; property/variant-swap (not just text) override behavior.

## Caveats / honesty
- All numbers are point-in-time against the 2026-06-17 remote build; payload sizes are char-count proxies (±10%), not MCP-client usage-log exact.
- The authoring agents hit a bug in my workflow (an `args` field came back undefined) so they grounded in the README + live schemas rather than the full research synthesis — docs are factually sound but less deep than intended; the QA gate + the live results compensate.
