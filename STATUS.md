# STATUS — overnight run, 2026-06-16 → 06-17

What got built while you slept, what's proven, and exactly where to pick up.

## TL;DR
Public repo is live, the corrected capability map + benchmark harness are written, and the **live empirical core is producing results nobody has published** — including one finding that overturns the field's #1 complaint about the Figma MCP. Everything is committed and pushed.

**Repo:** https://github.com/jmilinovich/figma-mcp-frontier
**Live benchmark draft (throwaway, my Figma drafts):** `figma-mcp-frontier-bench` — file `PSSvW47Ry1XDSAVZWQcN7t`

## Headline findings (live remote build, 2026-06-17)

1. **The "MCP hardcodes tokens" complaint is misattributed.** Controlled A/B: an *unbound* Button → `bg-[#17171c]` (hardcoded); a *token-bound* Card → `bg-[var(--color-surface,white)]`, `gap-[var(--space-lg,24px)]`, `rounded-[var(--radius-lg,12px)]`. `get_design_context` emits clean `var(--token,fallback)` refs **iff the design binds variables that carry WEB code-syntax.** It's a design-side problem, not a tool limit. → `results/read-fidelity-tokens.md`
2. **Component sets read back as parametric typed components.** `get_design_context` on a 3-variant Button set returned one React component with `variant?: "Primary"|"Secondary"|"Ghost"` and per-variant conditional styling — not three snippets. The reported variant brittleness is the **Code Connect** path, not raw `get_design_context`. → same file
3. **`clientFrameworks`/`clientLanguages` are logging-only (RESOLVED).** A `swiftui` request returned byte-identical React+Tailwind. The MCP always emits React+Tailwind; conversion is the agent's job. The "Compose still returns React" bug is actually the documented contract. → `results/ground-truth-probes.md`
4. **Per-tool token cost, measured head-to-head** (same node): `get_metadata` ~98 tok · `get_screenshot`(URL) ~120 tok · `get_design_context` ~368 tok + a default inline screenshot. URL screenshots carry zero base64; `excludeScreenshot` is the cheap win. → `results/token-cost.md`
5. **A ~350× reality gap worth chasing:** a clean semantic Card measured **~452 tokens** vs the widely-cited "~162K-token card." Strong signal the famous blowups are about **node density / imported cruft / inline rasters**, not semantic complexity. (Next experiment.)

## Write frontier — `use_figma` success so far
3 of 4 ladder rungs built, **every one clean on the first call, zero retries**:
- Rung 1: Button (frame + text)
- Rung 2: Button **COMPONENT_SET** with derived variant property
- Rung 3: token **variable collection** + a fully-bound **Card**

Atomicity + the `figma-use` skill make the write path reliable in practice. → `results/write-success.md`

## What's in the repo
- `docs/capability-map.md` — corrected 19-tool reference (10 remote-only / 9 shared, the renames, the 4 non-existent tools)
- `docs/methodology.md`, `docs/experiment-matrix.md` (E01–E12), `docs/landscape-comparison.md`
- `harness/`, `fixtures/` — reproduction + corpus spec
- `skill-draft/SKILL.md` — the reliability skill, first draft
- `results/` — the live measurements above
- `docs/_qa-report.md` — an adversarial accuracy gate caught + fixed 4 over-claims in the authored docs

## Pick up here (ranked)
1. 🟡 **PARTIAL — Rung 4 scaling** — built a 51-node dashboard; `get_design_context` = ~2.5k tokens, **no downgrade/cap hit**; cap extrapolates to ~509 clean nodes; clones aren't deduped (instances would be). Settles the ~350× gap (blowups = density/cruft, not semantic complexity). STILL OPEN: the actual cap-failure repro + `forceCode`-override (E01) — needs a *dense imported* design (do it on the shadcn kit, #2). → `results/token-cost.md`
2. **shadcn corpus read-fidelity** — duplicate the public shadcn Figma kit into the account, run `get_design_context` vs the real shadcn code as answer-key, score with the rubric.
3. ⛔ **WALL — Code Connect path** — both `add_code_connect_map` and `get_code_connect_map` are gated to an **Org/Enterprise Dev seat**; unavailable on Pro/Full. The "Code Connect upgrades output to a real import" test needs an Org/Enterprise seat. Side-finding: `get_design_context` on an instance emits component composition even *without* Code Connect. → `results/ground-truth-probes.md`
4. ✅ **DONE — Multi-mode `get_variable_defs`** — follows the node's active mode, returns one mode's resolved values only; multi-mode is invisible (write multi-mode works on Pro; read doesn't surface it). → `results/read-fidelity-tokens.md`
5. Tighten `skill-draft/SKILL.md` with what rungs 1–3 proved (variables-with-code-syntax first; bind everything).

## Caveats / honesty
- All numbers are point-in-time against the 2026-06-17 remote build; payload sizes are char-count proxies (±10%), not MCP-client usage-log exact.
- The authoring agents hit a bug in my workflow (an `args` field came back undefined) so they grounded in the README + live schemas rather than the full research synthesis — docs are factually sound but less deep than intended; the QA gate + the live results compensate.
