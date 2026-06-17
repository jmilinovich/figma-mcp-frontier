---
name: figma-write-reliably
description: >-
  DRAFT reliability protocol for the OFFICIAL remote Figma MCP (mcp.figma.com via
  the Claude Code plugin, mcp__plugin_figma_figma__*). Makes the non-deterministic
  use_figma write / round-trip pipeline repeatable: skill-gating, plan/file setup,
  one-wrapper-per-call section building (never reparent across calls), serialized
  (never parallel) library builds, small op-batches that return node IDs not
  payloads, font + color + image discipline, and per-call screenshot/metadata
  self-validation with atomic retry-after-fix. Use BEFORE any use_figma /
  generate_figma_design / upload_assets write, when a write "succeeded" but the
  canvas is wrong, when a build orphaned nodes, or when a library build is flaky.
allowed-tools:
  - mcp__plugin_figma_figma__whoami
  - mcp__plugin_figma_figma__create_new_file
  - mcp__plugin_figma_figma__use_figma
  - mcp__plugin_figma_figma__upload_assets
  - mcp__plugin_figma_figma__generate_figma_design
  - mcp__plugin_figma_figma__get_screenshot
  - mcp__plugin_figma_figma__get_metadata
  - mcp__plugin_figma_figma__get_variable_defs
  - mcp__plugin_figma_figma__search_design_system
  - Skill
---

# figma-write-reliably (DRAFT)

> **DRAFT — verified 2026-06-17 against the remote server (`mcp.figma.com`) via Anthropic's
> Claude Code plugin (`mcp__plugin_figma_figma__*`, 19 tools).** The rules below are distilled
> from the figma-mcp-frontier research synthesis + live tool schemas. The **"Proven at the canvas"**
> rules below are now **measured** against the live server (see `results/`); numeric ceilings (op
> count, payload size) remain working conventions, **not** measured limits — the benchmark
> (`harness/`, `docs/experiment-matrix.md`) will tighten them. Treat tagged confidence as such.

This is the **official remote** server, not the local Dev Mode server (`127.0.0.1:3845`) and not a
third-party server (Framelink/GLips `figma-context-mcp`, `cursor-talk-to-figma-mcp`). `use_figma`
runs **arbitrary JavaScript against the Figma Plugin API** — it is **atomic** (a thrown error rolls
back the whole script, zero partial changes → safe to retry) but **non-deterministic** (the model
writes fresh code each time). This protocol exists to tame the non-determinism, not the atomicity.

---

## Proven at the canvas (live 2026-06-17) — the write-time levers that decide read quality

These are no longer conventions — they were measured against the remote server (see `results/`). They are the highest-leverage moves, because **the quality and cost of the code you later read back is bought at write time.**

1. **Buy token fidelity: variables-with-code-syntax FIRST, then bind everything.** Build a variable collection, set explicit `scopes` AND a WEB **code-syntax** on each variable (`v.setVariableCodeSyntax("WEB", "var(--color-surface)")`), then bind every fill/stroke/cornerRadius/padding/gap/text-color to them. `get_design_context` then emits clean `var(--token,fallback)` refs (e.g. `bg-[var(--color-surface,white)]`). **Skip the binding and it hardcodes hex** (`bg-[#17171c]`) — that, not a tool defect, is the entire "the MCP hardcodes tokens" complaint. (Proven: rung-1 unbound vs rung-3 bound → `results/read-fidelity-tokens.md`.)
2. **Use component INSTANCES, not clones, for anything repeated.** `get_design_context` componentizes true instances (one component def + short usages — DRY and cheap) but emits **N full copies of the markup for clones**. Measured ~3–4× output reduction on a 12-item grid; the gap grows with count. Make it a component, instantiate it. (Proven: rung-2 instance vs rung-4 clones → `results/token-cost.md`.)
3. **`combineAsVariants` pays off on read.** A proper component set makes `get_design_context` return a **single parametric, typed** component (`variant?: "Primary"|"Secondary"|"Ghost"` + per-variant conditional classes), not snippets. Name variant children `"Prop=Value"` so the property is derived. (Proven: rung 2.)
4. **Clean designs read cheap — the 25k cap is a *density* problem.** ~197 chars/node measured; a 51-node screen ≈ 2.5k tokens, no downgrade. The famous 162K/351k-token blowups are imported/vector/raster-dense designs, not semantic complexity. So: build clean, scope tightly, and reach for `get_metadata` first (it stays ~3–4× lighter and the gap widens with size). (Proven: rung 4 → `results/token-cost.md`.)
5. **Don't expect framework retargeting.** `clientFrameworks`/`clientLanguages` are **logging-only** — output is always React+Tailwind; converting to Vue/SwiftUI/Compose is the agent's job downstream. (Proven → `results/ground-truth-probes.md`.)
6. **Code Connect needs an Org/Enterprise Dev seat.** Both `get_code_connect_map` and `add_code_connect_map` are gated there — unavailable on Pro/Full. Don't build a pipeline that assumes it; the local component-composition `get_design_context` gives is the fallback. (Proven this build.)
7. **Multi-mode: you can write it, you can't fully read it.** Adding a Dark mode works on Pro, but `get_variable_defs` returns only the node's **active** mode's resolved values — never the multi-mode definition. For full theming, read twice under explicit modes or use the REST API. (Proven → `results/read-fidelity-tokens.md`.)

> **Reliability signal so far:** 4/4 ladder rungs (Button, variant set, token-bound Card, 51-node screen) built **clean on the first call, zero retries** when following the protocol below. Atomicity + `figma-use` discipline is what makes that hold.

---

## The protocol (run top to bottom)

### 0. Gate on the skill — every time, before the first `use_figma`
Load `figma-use` (the plugin's HOW-to-call-the-API skill) **before** any `use_figma` call. The tool
description itself mandates it; skipping it is the single most common cause of "ran clean but the
canvas is wrong." For component/variant/token builds, also load `figma-generate-library` (it teaches
WHAT to build and in what order). Pass the loaded skills back via the `skillNames` param
(e.g. `"figma-use,figma-generate-library"`) for logging.

```
Skill(figma-use)                         # always
Skill(figma-generate-library)            # when building a design system / components
```

### 1. Resolve the plan key — `whoami` first
`create_new_file` requires a `planKey` matching `^(team|organization)::\d+$`. Call `whoami`, read the
returned plans. One plan → use its `key` verbatim. Multiple → **ask the user**; do not guess. Never
hand-fabricate a key.

### 2. Create a throwaway DRAFT file — build there, not in the real file
`create_new_file({ editorType: "design", fileName: "<scratch>", planKey })` → keep the returned
`file_key` / URL. Drafts have **no Full-seat gate** (the `use_figma` write beta is restricted to Full
seats for non-draft files; Dev seats are read-only outside drafts) and **zero blast radius**. Promote
to the real file only once the draft validates. Confidence: high (matches seat-gating in README).

### 3. Plan the build as a node tree, then build **section by section**
Decompose the target into a tree of sections (e.g. page → header / hero / grid / footer). Each
section = **one `use_figma` call** that creates that subtree **inside a single wrapper node and
appends it in the same call**.

> **The cardinal rule: never reparent across calls — it orphans.** (Confidence: high; flagged in
> synthesis.) Do **not** create a node in call A and `appendChild` it under a parent created in
> call B. Nodes created and not parented within their own call get orphaned / land at page root /
> are lost on the next call's view of the tree. Create-and-parent **atomically, within one call.**
> If a later section must attach to an earlier one, re-fetch the earlier parent by its **returned
> node ID** at the top of the new call and append into it there.

### 4. Keep each call small and return IDs, not payloads
- **≤ ~8–10 logical operations per call.** (Convention, not a measured limit — `E0x` in the
  experiment matrix binary-searches the true ceiling.) Smaller calls = more deterministic, cheaper
  to validate, cheaper to retry.
- **The `code` field is hard-capped at 50,000 chars** (live schema `maxLength` — verified).
  Tool **output** (what your script returns) is effectively capped around **~20 KB** — exceeding it
  fails the call (convention, **not** a measured limit; the read-side analog is the 25,000-token
  per-tool cap, which *fails the call outright rather than truncating* — see `harness/`).
- **Return only what the next step needs: node IDs, names, counts.** End each script with a compact
  `console.log(JSON.stringify({ root: frame.id, children: [...] }))`. **Never** serialize and return
  whole node trees or `exportAsync` blobs — that blows the output cap and wastes context.

### 5. Serialize library builds — **never parallelize `use_figma`**
For any multi-call build (especially variables → styles → components → variants), issue calls
**strictly one at a time, awaiting each.** **Parallel `use_figma` calls corrupt library builds**
(racing writes to the same document / variable collection). (Confidence: high; flagged in synthesis +
README method note.) One in-flight write at a time, period.

### 6. Asset, font, and color discipline (the silent failure cluster)
- **Colors are 0–1 floats**, not 0–255 and not hex. `{ r: 0.2, g: 0.4, b: 0.9 }`. Passing 0–255 or a
  hex string silently produces black / wrong fills.
- **Fonts: `await figma.loadFontAsync(...)` before setting any text** (`characters`, `fontName`,
  `fontSize`). **Google Fonts only** — there is **no custom-font support**. Watch the casing gotchas:
  Inter uses `"Semi Bold"` / `"Extra Bold"` (with a space), not `"SemiBold"`.
- **No external image URLs inside `use_figma`** — `figma.createImage(url)` against a remote URL fails;
  the Plugin API sandbox cannot fetch arbitrary URLs. To place a raster:
  - **`upload_assets`** — request N upload URLs (`count` 1–5), POST raw bytes with the correct
    `Content-Type`; with `nodeId` (count=1) it sets the image as a fill on that node, without it
    creates frames. PNG/JPG/GIF/WebP, **max 10 MB**. **SVG is NOT supported here** — for SVG call
    `figma.createNodeFromSvg(...)` inside `use_figma`.
  - **`generate_figma_design`** captures a web page/HTML into an *existing design file* (two-phase:
    call with `fileKey` and no `captureId` → get a capture script + `captureId`, then poll that
    `captureId` every 5s up to 10 times until `completed`). It lands a **flat dead raster** node — no
    component instances, no variant props, no variable bindings (staff-acknowledged #1 write-frontier
    gap). Per the tool's own contract, use it as a **layout reference**, then rebuild from real
    components with `use_figma` and **delete the raster** — do not treat its output as the deliverable.

### 7. Validate every call — self-check, don't trust the return string
A clean tool return means the JS didn't throw, **not** that the canvas looks right. After each
section call, validate the **node ID it returned**:
- **`get_screenshot({ nodeId, fileKey })`** — prefer the returned **URL + curl** over
  `enableBase64Response` (base64 is far more expensive in context; only use it when you can't fetch
  URLs). Eyeball: did it render, is it positioned/sized, are fills/fonts right.
- **`get_metadata({ fileKey, nodeId })`** — confirm the subtree's structure, that children are
  parented where intended (catches silent orphaning from step 3), names, positions, sizes.
- For tokens/variables, **`get_variable_defs`**; to find existing components to reuse,
  **`search_design_system`** before creating duplicates.

> **OPEN (expertiseGaps — do not assume):** remote `get_metadata` has a reported
> "instruction-only / first-design-only" pathology where it may describe only the first design or
> return guidance instead of the full structure. Until `docs/experiment-matrix.md` resolves it,
> **cross-check structural validation with a `get_screenshot`** rather than trusting `get_metadata`
> alone.

### 8. Atomic retry-after-fix (not blind retry)
Because `use_figma` is atomic, a failed/wrong section left **zero** partial state — the document is
clean. So on a bad validation:
1. Read the error / diff the screenshot against intent.
2. **Fix the script** (the most common real causes: color scale, unloaded font, cross-call reparent,
   external image URL, output over ~20 KB, > ~10 ops).
3. Re-run **that one section only**. Do not pile a "fix" call on top of a half-built section — rebuild
   the section atomically.
4. If two corrected attempts fail, **split the section into smaller calls** (step 4) before trying a
   third — non-determinism compounds with op count.

---

## Pre-flight checklist (copy before any write)

- [ ] `figma-use` loaded (+ `figma-generate-library` if building a system); `skillNames` set
- [ ] `whoami` resolved a real `planKey` (`team::…` / `organization::…`)
- [ ] Building in a **throwaway draft** file (no Full-seat gate, zero blast radius)
- [ ] Build decomposed into a section tree; each section is **one create-and-parent call**
- [ ] **No** cross-call reparenting (re-fetch parents by returned node ID instead)
- [ ] Calls **serialized** — never parallel `use_figma`
- [ ] ≤ ~8–10 ops/call; `code` ≤ 50 000 chars; return **IDs not payloads** (≤ ~20 KB output)
- [ ] Colors 0–1; `loadFontAsync` before text; **Google Fonts only**
- [ ] Token fidelity bought at write time: variables w/ WEB code-syntax, then **bind everything**; repeated structures are **instances, not clones**
- [ ] No external image URLs — `upload_assets` / `generate_figma_design` (capture→rebuild→delete); SVG via `createNodeFromSvg`
- [ ] Each call validated via `get_screenshot` (URL+curl) and/or `get_metadata` on the returned ID
- [ ] On failure: diagnose → fix script → re-run that section atomically → split if still failing

---

## Common failure → cause → fix

| Symptom | Likely cause | Fix |
|---|---|---|
| "Ran clean," canvas blank/wrong | `figma-use` not loaded; wrong assumptions about API | Load `figma-use` first; re-derive the call |
| Nodes scattered at page root | Reparented across calls (step 3) | Create + `appendChild` in the **same** call; re-fetch parent by ID |
| Fills come out black | Colors passed as 0–255 / hex | Use 0–1 floats |
| Text empty / font error | Forgot `loadFontAsync`, or non-Google / mis-cased font | `await loadFontAsync`; Google Fonts; `"Semi Bold"` not `"SemiBold"` |
| Image never appears | External URL inside `use_figma` | `upload_assets` or `generate_figma_design` (capture→rebuild→delete); SVG → `createNodeFromSvg` |
| Generated code hardcodes hex/px | Values not bound to variables (or variables lack WEB code-syntax) | Bind fills/radius/spacing/text to code-synced variables — fidelity is bought at write time |
| Read-back code is N copies of one block | Built with clones, not instances | Make it a component, instantiate it — `get_design_context` componentizes instances |
| Call truncated / failed on big return | Output over ~20 KB | Return IDs/counts only, not node trees |
| Library build randomly corrupts | Parallel `use_figma` calls | Serialize — one in-flight write at a time |
| Validation looks fine but structure off | `get_metadata` first-design-only pathology (**OPEN**) | Cross-check with `get_screenshot`; see experiment matrix |

---

## Resolved this build (2026-06-17) — promoted from convention to measured
- ✅ Token fidelity, instances-vs-clones, `combineAsVariants` read shape, multi-mode read limit, Code Connect seat gate, framework-param logging-only — see the "Proven at the canvas" section + `results/`.
- ✅ `get_design_context` shows **no downgrade/cap at 51 clean nodes** (~2.5k tokens); the 25k cap extrapolates to ~509 clean nodes — confirming the blowups are a density/cruft problem.
- ✅ `get_metadata` returned full structure for a 51-node, multi-card tree (no "first-design-only" pathology observed on a single multi-child design this build).

## Still to be tightened by the benchmark (`harness/`, `docs/experiment-matrix.md`)
- True single-call ceiling: ops, node count, and the real output-size limit (the ~10 ops / ~20 KB figures remain conventions; rung 4 didn't stress them).
- The definitive cap-failure repro + whether `forceCode` overrides the silent downgrade — needs a **dense imported** design (e.g. the shadcn kit), not clean synthetic nodes.
- Whether the `get_metadata` "first-design-only" pathology appears on a genuinely **multi-design page** (untested; single multi-child design was fine).
- Exact `use_figma` non-determinism rate over a large library build (4/4 clean so far is a small sample).

*Point-in-time against a fast-moving target. Re-verify the dated claims before relying on them.*
