# Write success-rate — `use_figma` — live build 2026-06-17

Each rung of the controlled complexity ladder, built into draft `PSSvW47Ry1XDSAVZWQcN7t` via `use_figma` (skill `figma-use` loaded first, per protocol). Metric: did the script execute **atomically on the first call** (clean), or did it take N retries? Plus a note on result correctness.

| Rung | What | Result | Calls / retries | Node IDs | Notes |
|---|---|---|---:|---|---|
| 1 | Single Primary Button (auto-layout frame + Inter Medium text) | ✅ PASS | 1 / 0 | frame `1:2`, text `1:3` | Clean first-call success. `createAutoLayout` + text-child HUG after append worked as the skill prescribes. |
| 2 | Button COMPONENT_SET — `Variant` property (Primary/Secondary/Ghost) via `combineAsVariants` | ✅ PASS | 1 / 0 | set `4:8`, variants `4:2`/`4:4`/`4:6` | Clean first-call. `combineAsVariants` derived `variantGroupProperties = {Variant:[Primary,Secondary,Ghost]}` atomically. `get_design_context` on the set → a single **parametric typed React component** (`variant?: "Primary"\|"Secondary"\|"Ghost"`) with per-variant conditional classes — see read-fidelity-tokens.md. |
| 3 | Tokenized Card (variable collection `2:2` → bound fills/stroke/radius/padding/gap + text colors) | ✅ PASS | 2 / 0 | coll `2:2`, card `3:2`, title `3:3`, body `3:4` | Built in 2 clean calls (variables first, then the bound Card). **Headline payoff → results/read-fidelity-tokens.md:** binding code-synced variables makes `get_design_context` emit `var(--token,fallback)` refs instead of hardcoded hex. |
| 4 | Dashboard screen — header + wrapping grid of 12 stat cards (~51 nodes) | ✅ PASS | 2 / 0 | screen `9:3`, grid `9:5`, cards `9:6`+`10:3..10:43` | Built clean (base card + 11 clones). `layoutWrap='WRAP'` grid worked first try. Read at scale → ~2.5k tokens, **no downgrade/cap hit at 51 nodes** (cap extrapolates to ~509 clean nodes); clones emit as repeated inline divs (no dedup) → see token-cost.md. `forceCode`/downgrade definitive test deferred to a dense import. |

## Observations so far

- **Atomicity confirmed as a real safety property.** The skill's claim that a failed `use_figma` makes zero changes is the backstop that makes the write frontier tolerable; rung 1 needed no retry so it wasn't exercised here, but it's the design contract the harness relies on.
- **The `figma-use` skill materially de-risks the write.** Rung 1's script followed the skill's rules verbatim (colors 0–1, `loadFontAsync` before text, set `layoutSizingHorizontal='HUG'` only *after* `appendChild`, position away from 0,0, return all node IDs) and succeeded first try. The skill encodes exactly the traps that otherwise cause silent failures.

## Method

- One `use_figma` call per rung where possible; ≤~10 logical ops/call (skill rule). Larger rungs split across calls.
- **Never parallelize `use_figma`** for these builds (parallel calls corrupt library/variant state — skill + research agree).
- Build-date stamped; draft-file-only (no Full-seat-gated real files touched).
