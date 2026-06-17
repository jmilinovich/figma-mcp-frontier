# Experiment Matrix

**The executable spec the live benchmark follows.** Every open question about the official remote Figma MCP — drawn from the research synthesis `expertiseGaps` and the bug-repros in `topIssues` — is turned here into a concrete, runnable experiment.

> Subject under test: the **remote** Figma MCP (`mcp.figma.com`) via Anthropic's Claude Code plugin (`mcp__plugin_figma_figma__*`), **19 tools**, tool schemas introspected live **2026-06-17**.
> This file is point-in-time. Re-run against the current build and re-stamp `verifiedOn` per row.

## How to read this file

Each experiment has a fixed shape:

| Field | Meaning |
|-------|---------|
| **ID** | `E01`..`E12`. Cite this ID in `results/` and `findings.md`. |
| **Question / hypothesis** | The open question, phrased so a result can confirm or refute it. |
| **Tools + exact params** | Real tool names (`mcp__plugin_figma_figma__*`) and the exact param keys from the live schema. |
| **Procedure** | Numbered, deterministic steps. A second operator should get the same trace. |
| **Confirm / refute signal** | The observable that decides the outcome. No interpretation latitude. |
| **Result** | Left blank: `PASS` / `FAIL` / `NEEDS-RERUN` + date + build. Fill in `results/`. |

### Standing conventions (apply to every experiment)

- **Fixtures.** All reads run against the **complexity ladder** + **shadcn corpus** in `fixtures/` (see `fixtures/` spec). Where an experiment needs "one representative component," use the **fixture tagged `repr-card`** — a shadcn-style Card with: 1 component instance, ≥3 bound variables (color + spacing + radius), 1 text layer, 1 icon. Record its `fileKey` + `nodeId` in the result so the row is reproducible.
- **Writes** run only into **throwaway draft files** (no Full-seat gate, zero blast radius). One `use_figma` call sequence at a time. **Never parallelize `use_figma`** — parallel writes corrupt library builds.
- **Token accounting.** "Token cost" = the per-tool-call token count the Claude Code client reports for the tool *result* (not the request). When the client does not surface a number, fall back to `wc -c` on the raw result payload ÷ 4 as an estimate and **flag it as estimated**.
- **Atomicity check.** After any `use_figma` that errors, re-read the target with `get_metadata`; a true-atomic failure leaves the node graph unchanged.
- **Isolation.** Fresh node target per experiment where state could leak. Don't reuse a node that a prior write mutated.

---

## Results so far (live build 2026-06-17)

Status of each experiment after the first live run. Detail + raw payloads in `results/`.

| ID | Status | Finding (short) | Detail |
|---|---|---|---|
| E01 `forceCode` | 🟡 PARTIAL | No-op on small nodes (byte-identical output). The size-downgrade-override case is untested — needs a node past the threshold. | `results/ground-truth-probes.md` |
| E02 single-call ceiling | 🟡 PARTIAL | 51-node screen built clean in 2 calls, 0 errors; `get_design_context` hit no cap at 51 nodes. True op/node/output/time ceilings still unmeasured. | `results/write-success.md`, `results/token-cost.md` |
| E03 `setReactionsAsync` | ✅ RESOLVED | Executes + persists; reactions read back via `node.reactions`. Prototyping is scriptable. | `results/ground-truth-probes.md` |
| E04 effect params (Noise/Texture/Glass) | ✅ RESOLVED | All settable (Texture/Glass clean; NOISE needs `blendMode` dropped — a `.d.ts`-vs-runtime drift). Newer effects are scriptable. | `results/ground-truth-probes.md` |
| E05 Feb-2026 Code Connect bugs | ⛔ BLOCKED | Code Connect (read **and** write) is gated to an Org/Enterprise Dev seat — untestable on Pro/Full. | `results/ground-truth-probes.md` |
| E06 `get_metadata` first-design-only | ✅ RESOLVED | NOT reproduced: returned **all 10 top-level designs** on the page, fully nested. Likely desktop-specific or since-fixed. | `results/ground-truth-probes.md` |
| E07 per-tool token cost | 🟢 DONE (rung 1) | `get_metadata` ~98 · `get_screenshot`(URL) ~120 · `get_design_context` ~368 tok + a default inline screenshot. Scales to ~2.5k tok at 51 nodes. (N≥3 medians still to add.) | `results/token-cost.md` |
| E08 `clientFrameworks`/`clientLanguages` | ✅ RESOLVED | **Logging-only** — output is always React+Tailwind; a SwiftUI request returned byte-identical React. | `results/ground-truth-probes.md` |
| E09 instance override values | ✅ RESOLVED (text) | `get_design_context` returns the override ("Submit now"), not the base — refutes the "returns base values" worry. Variant-swap overrides untested. | `results/ground-truth-probes.md` |
| E10 atomicity | ✅ RESOLVED | Both thrown-JS and API-level failures leave **zero** nodes — atomic, safe to retry. | `results/ground-truth-probes.md` |
| E11 `disableCodeConnect` | ⛔ BLOCKED | Moot on this account — Code Connect is seat-gated off entirely. | `results/ground-truth-probes.md` |
| E12 `excludeScreenshot` cost | 🟡 PARTIAL | Confirmed it suppresses the inline image (text identical with/without); precise screenshot-token delta not yet isolated. | `results/token-cost.md` |

**Bonus findings — proven but not originally enumerated here** (→ `results/read-fidelity-tokens.md`): token fidelity is bought at write time (bound code-synced variables → `var(--token,fallback)`; unbound → hardcoded hex); `get_design_context` componentizes true **instances** but emits N copies for **clones**; a component **set** reads back as one parametric typed component; **multi-mode** `get_variable_defs` returns only the node's active mode.

---

## E01 — Does `get_design_context` expose and honor `forceCode`, and what does it toggle?

**Question / hypothesis.** The live schema exposes `forceCode: boolean` ("Whether code should always be returned, instead of returning just metadata if the output size is too large"). Hypothesis: on a node whose default `get_design_context` response is *demoted to metadata* because the code payload is too large, setting `forceCode: true` flips the response back to full React+Tailwind code (and risks blowing the 25,000-token per-tool cap). Open sub-question: what is the **demotion threshold** — at what output size does the default (no `forceCode`) silently return metadata instead of code?

**Tools + exact params.**
- `mcp__plugin_figma_figma__get_design_context` with `{ fileKey, nodeId }` (default), then `{ fileKey, nodeId, forceCode: true }`.
- `mcp__plugin_figma_figma__get_metadata` to confirm the node's structure between calls.

**Procedure.**
1. Pick a **large** node — a full screen / dense frame from the ladder's top rung (target ≥ a few hundred descendant nodes per `get_metadata`).
2. Call `get_design_context` with **no** `forceCode`. Record: did the response contain a React/Tailwind **code string**, or a **metadata/structural** payload (the demoted form)? Record token count.
3. Call `get_design_context` again on the **same node** with `forceCode: true`. Record response type + token count.
4. If step 2 already returned code (no demotion observed), **binary-search node size**: repeat steps 2–3 on progressively larger nodes (more descendants / more bound styles) until the default response demotes to metadata, to locate the demotion threshold. Record the node size (descendant count + default token count) at the flip.
5. Diff the `forceCode:true` payload against the demoted default: confirm `forceCode` is what restores the code, not some other field.

**Confirm / refute signal.**
- **Confirms** "exposes + honors": there exists a node where default → metadata-only and `forceCode:true` → full code string. Record the threshold token/size at the flip.
- **Refutes**: `forceCode:true` yields an identical payload to default for all tested sizes (param is inert), OR the param is rejected/ignored by the live server.
- Secondary capture: the token count of a `forceCode:true` response on a node that blows the 25k cap (does it hard-fail the call, per the documented 351,378-token worst case, or truncate?).

**Result:** _____  (date / build: _____)

---

## E02 — True single-call `use_figma` ceiling (node count + payload), beyond the conventions

**Question / hypothesis.** The community conventions are "≤ ~10 ops per call" and "keep result output < ~20kb." The **hard** limit visible in the live schema is `code.maxLength = 50000` characters (input). Open question: what actually caps a *single* `use_figma` call — the 50k input char limit, an output-size limit, a node-creation count limit, or an execution-time limit — and where is each ceiling? Hypothesis: the binding constraint is reached well before 50k input chars, via either node-count or wall-clock, not the documented "10 ops."

**Tools + exact params.**
- `mcp__plugin_figma_figma__use_figma` with `{ fileKey, code, description }`. (`code.maxLength` = 50000; `description.maxLength` = 2000.)
- `mcp__plugin_figma_figma__get_metadata` to count nodes actually created after each call.

**Procedure** (binary search, into a throwaway draft):
1. **Input-char axis.** Generate a `code` string that creates `N` trivial rects in a loop. The *generator* is short, so test whether a literally large `code` body (padded toward 50,000 chars with a big inline data array) is accepted or rejected. Binary-search the char count between the last accepted and first rejected size. Record the rejection error string.
2. **Node-count axis.** With a short loop body, set `N` = 50 → 100 → 200 → 500 → 1000 → 2000 created nodes. After each call, `get_metadata` the parent and count actual children created. Binary-search between the largest `N` that fully succeeds and the smallest `N` that errors or partially completes. Record the ceiling and the failure mode (error vs partial vs timeout).
3. **Output-size axis.** Make a call whose *return value* is large (e.g., `code` ends by returning a big serialized object). Binary-search the returned-payload size at which the call fails or is truncated. Compare to the ~20kb convention.
4. **Op-count axis.** Hold node count low but chain many distinct Plugin API operations (create, setFill, setEffect, reorder, group, …). Find whether a count-of-distinct-ops limit exists independent of node count. Test the "≤10 ops" convention directly.
5. For every failure, run the **atomicity check**: re-read with `get_metadata`. Note whether a failed call left partial nodes (refutes atomicity) or none (confirms it).

**Confirm / refute signal.**
- Four numbers, each with its failure mode: max input chars, max nodes/call, max return-payload size, max distinct ops/call. The **smallest binding** one in practice is the real single-call ceiling.
- **Atomicity** confirmed iff every errored call left zero nodes on re-read.
- Refutes the "10 ops" convention if calls with ≫10 ops succeed reliably.

**Result:** _____  (date / build: _____)

---

## E03 — Does `setReactionsAsync` execute under `use_figma`?

**Question / hypothesis.** `use_figma` runs arbitrary JS against the Figma Plugin API. Open question: does the **prototyping** surface — specifically `node.setReactionsAsync(...)` (click → navigate / overlay / etc.) — actually execute in the MCP's plugin sandbox, or is the reaction API unavailable/no-op in this context? Hypothesis: reactions can be *set* but the result is not observable through any read tool (no read tool surfaces prototype reactions), so the test must verify via a programmatic read-back inside a second `use_figma` call.

**Tools + exact params.**
- `mcp__plugin_figma_figma__use_figma` with `{ fileKey, code, description }`. Load `/figma-use` first (mandatory).
- Read-back: a second `use_figma` call that reads `node.reactions` and **returns** it as the call's result (since no dedicated read tool exposes reactions).

**Procedure** (throwaway draft):
1. Create two frames `A` and `B` via `use_figma`. Capture their node IDs (return them from the call).
2. In a **separate** `use_figma` call, on frame `A` call `await A.setReactionsAsync([{ trigger: { type: 'ON_CLICK' }, actions: [{ type: 'NODE', destinationId: B.id, navigation: 'NAVIGATE', transition: null }] }])`. Have the code `return { ok: true, error: null }` or catch and return the error string.
3. In a **third** `use_figma` call, read `figma.getNodeById(aId).reactions` and **return** the serialized reactions array.
4. Inspect the returned reactions: is the ON_CLICK → NAVIGATE-to-B reaction present and well-formed?

**Confirm / refute signal.**
- **Confirms**: step 2 returns `ok:true` AND step 3's returned `reactions` contains the navigate action targeting `B`.
- **Refutes**: step 2 throws (capture the exact error — e.g. `setReactionsAsync is not a function`, permissions, or "not available in this context"), OR step 3 returns an empty `reactions` array despite step 2 reporting success (set silently dropped).
- **NEEDS-RERUN**: step 2 succeeds but step 3 errors for unrelated reasons.

**Result:** _____  (date / build: _____)

---

## E04 — Are Noise / Texture / Glass effect params settable via `use_figma`?

**Question / hypothesis.** Newer Figma effect types (Noise, Texture, and the "Glass"/progressive-blur family) have parameters that may or may not be exposed/writable through the Plugin API surface the MCP runs against. Open question: which of these effect types can be **set programmatically** through `use_figma`'s `effects` array, with which params, and which are read-only / UI-only / rejected? Hypothesis: classic effects (DROP_SHADOW, INNER_SHADOW, LAYER_BLUR, BACKGROUND_BLUR) set cleanly; the newer Noise/Texture/Glass types either error on the `type` value or accept it but drop the type-specific params.

**Tools + exact params.**
- `mcp__plugin_figma_figma__use_figma` with `{ fileKey, code, description }`.
- Read-back via `use_figma` returning `node.effects` (programmatic), plus `mcp__plugin_figma_figma__get_screenshot` `{ fileKey, nodeId, maxDimension: 2048 }` for visual confirmation.

**Procedure** (throwaway draft, one effect type per call):
1. Create a base rect. For each effect type below, in its own `use_figma` call set `node.effects = [<effect spec>]` and `return node.effects` (the round-tripped value).
   - `NOISE` (test its params: `noiseType`, `density`/`opacity`, `noiseSize`, `secondaryColor` — whatever the current Plugin API names are).
   - `TEXTURE` (params: `noiseSize`, `radius`, `clipToShape`, etc.).
   - The glass / progressive-blur family (`GLASS` if present, else `BACKGROUND_BLUR` with the progressive params).
   - Control: `DROP_SHADOW` with full params (color, offset, radius, spread) — must round-trip cleanly.
2. For each: compare the **returned** `effects` to what was set. Did the `type` survive? Did each type-specific param survive (settable) or get dropped/defaulted (not settable)?
3. `get_screenshot` the node at `maxDimension: 2048`; visually confirm the effect actually rendered (a param can round-trip in data but not render, or vice versa).
4. Record the exact Plugin API param names that worked vs were rejected — this is the deliverable for the capability map.

**Confirm / refute signal.**
- Per effect type: **settable** iff the call succeeds, the returned `effects` preserves the type-specific params, AND the screenshot shows the effect. **Not settable** iff the call errors on `type`, OR the params are silently dropped/defaulted in the round-trip, OR they round-trip but don't render.
- Control `DROP_SHADOW` must PASS for the experiment to be valid (else the harness, not the effect API, is at fault → NEEDS-RERUN).

**Result:** _____  (date / build: _____)

---

## E05 — Are the Feb-2026 Code Connect bugs still live on this build?

**Question / hypothesis.** Three Code Connect read defects were reported ~Feb 2026: (a) **snippets dropped on Component Sets** (the variant container returns no Code Connect snippet even when its variants are mapped); (b) **base-variant tokens** mis-resolved (the default/base variant returns wrong or empty token bindings); (c) **missing instance overrides** in the Code Connect output. Open question: are any/all still reproducible on the 2026-06-17 build? Hypothesis: at least one is still live (Code Connect read fidelity is a recurring `topIssues` theme).

**Tools + exact params.**
- `mcp__plugin_figma_figma__get_code_connect_map` with `{ fileKey, nodeId }` (and `codeConnectLabel` if multiple mappings exist).
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId }` (Code Connect-aware; do **not** set `disableCodeConnect`).
- `mcp__plugin_figma_figma__get_metadata` to identify the Component Set node vs its variant children.

**Procedure** (against a fixture with a **mapped Component Set**, e.g. a Button with variant + state variants, Code Connect attached):
1. `get_metadata` the fixture → identify the **Component Set** node ID, the **base/default variant** node ID, and a **component instance with overrides** node ID.
2. **(a) Component Set drop.** `get_code_connect_map` on the Component Set node. Then `get_design_context` on it. Signal: is a Code Connect snippet returned for the set itself? (Bug = no snippet on the set despite variants being mapped.)
3. **(b) Base-variant tokens.** `get_design_context` on the base/default variant; `get_variable_defs` on the same. Compare returned token bindings to the variant's actual bound variables (cross-check via a `use_figma` read-back of `node.boundVariables`). Bug = base variant returns empty/wrong tokens while a non-base variant returns correct ones.
4. **(c) Instance overrides.** On an instance that overrides text/props vs its main component, `get_design_context`. Bug = the override values are absent from the output (main-component defaults shown instead). (This overlaps E12 — share the fixture + reading.)
5. For each of (a)/(b)/(c): record reproduced / not-reproduced, with the exact returned payload.

**Confirm / refute signal.**
- Per defect: **still live** iff the buggy behavior reproduces exactly as described; **fixed** iff the correct value is returned. Stamp each independently — they can diverge.
- This row produces a 3-way verdict, not one PASS/FAIL. Record `(a)=?, (b)=?, (c)=?`.

**Result:** (a) _____ | (b) _____ | (c) _____  (date / build: _____)

---

## E06 — Remote `get_metadata` "instruction-only / first-design-only" pathology

**Question / hypothesis.** Reported pathology: on the **remote** server, `get_metadata` sometimes returns an *instruction string* (telling the agent what to do) instead of the XML structural dump, and/or only returns the **first** design/frame on a page rather than the full set. Open question: is this reproducible, and what triggers it — calling with no `nodeId` (page-list mode) vs a page `nodeId` vs a frame `nodeId`? Hypothesis: the pathology is tied to the page-level call (page `nodeId` like `0:1` or omitted `nodeId`), where the server returns guidance/first-node rather than enumerating all top-level frames.

**Tools + exact params.**
- `mcp__plugin_figma_figma__get_metadata` with three variants: `{ fileKey }` (omit `nodeId` → documented page-list mode), `{ fileKey, nodeId: <page id, e.g. 0:1> }`, and `{ fileKey, nodeId: <a specific frame> }`.

**Procedure** (against a file with a page containing **multiple** top-level frames, ≥3):
1. Call `get_metadata` with **no `nodeId`**. Expect: list of top-level pages (guid + name). Record whether it returned a page list, an instruction string, or XML.
2. Call `get_metadata` with the **page** `nodeId` (`0:1`). Expect: XML dump of the page's children. Record: did it enumerate **all** top-level frames, only the **first**, or return an instruction/guidance string?
3. Call `get_metadata` with a **specific frame** `nodeId`. Record response type + completeness.
4. Repeat steps 1–3 three times to check for **non-determinism** (the pathology may be intermittent).
5. Cross-check completeness: count top-level frames via a `use_figma` read-back (`figma.currentPage.children.length`) and compare to what step 2 enumerated.

**Confirm / refute signal.**
- **Pathology confirmed** iff a page-level `get_metadata` returns an instruction/guidance string instead of structure, OR enumerates only the first frame when ≥2 exist (verified against the `use_figma` child count).
- **Refuted** iff all three call shapes return complete, well-formed structural data on every repeat.
- Record determinism: N reproductions out of N attempts.

**Result:** _____  (date / build: _____)

---

## E07 — Per-call token cost head-to-head on one representative component

**Question / hypothesis.** The synthesis lacks controlled per-tool token numbers. Open question: on **one** representative component, what is the per-call token cost of each read path, ranked? Hypothesis (to verify, not assume): `get_metadata` is cheapest, `get_screenshot` via **URL** is far cheaper than `get_screenshot` **base64**, and `get_design_context` is the most expensive (and the one at risk of the 25k cap).

**Tools + exact params (all on the same `repr-card` node):**
- `mcp__plugin_figma_figma__get_metadata` `{ fileKey, nodeId }`.
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId }` (default), and a second run with `{ ..., excludeScreenshot: true }` to isolate code-vs-screenshot cost.
- `mcp__plugin_figma_figma__get_screenshot` `{ fileKey, nodeId, maxDimension: 1024 }` (URL/default, `enableBase64Response` omitted = false).
- `mcp__plugin_figma_figma__get_screenshot` `{ fileKey, nodeId, maxDimension: 1024, enableBase64Response: true }` (base64 inline).
- `mcp__plugin_figma_figma__get_variable_defs` `{ fileKey, nodeId }`.

**Procedure.**
1. Fix the target = `repr-card` fixture. Record its `fileKey`/`nodeId`.
2. Call each tool/variant **once**, in the order above. For each, record the client-reported result token count (or estimated `bytes/4`, flagged).
3. Repeat the full set **3×** to get a median per tool (token counts can vary slightly with screenshot encoding).
4. Build the ranking table. Compute the **base64 vs URL multiplier** for `get_screenshot`, and the **screenshot share** of a default `get_design_context` (default minus `excludeScreenshot:true`).
5. Note any call that **exceeds 25,000 tokens** (per-tool cap) on this single component — that itself is a finding.

**Confirm / refute signal.**
- Output is the ranked table (median tokens per call) + two derived ratios (base64÷URL, screenshot share of `get_design_context`). No PASS/FAIL — this is a **measurement**, marked `DONE` when the table is filled with N≥3 medians.
- Flags any hypothesis-breaking surprise (e.g., URL screenshot costing more than metadata, or `get_design_context` on a single card already blowing 25k).

**Result table (fill in):**

| Tool + params | Run1 | Run2 | Run3 | Median | Notes |
|---|---|---|---|---|---|
| `get_metadata` |  |  |  |  |  |
| `get_design_context` (default) |  |  |  |  |  |
| `get_design_context` (excludeScreenshot) |  |  |  |  |  |
| `get_screenshot` (URL) |  |  |  |  |  |
| `get_screenshot` (base64) |  |  |  |  |  |
| `get_variable_defs` |  |  |  |  |  |

Derived: base64÷URL = _____ · screenshot share of `get_design_context` = _____ · any call > 25k tokens? _____
**Result:** _____  (date / build: _____)

---

## E08 — Does `get_design_context` honor `clientFrameworks` / `clientLanguages` overrides?

**Question / hypothesis.** The schema says `clientFrameworks` and `clientLanguages` are "used for logging purposes." Open question: are they **purely** telemetry, or do they actually change the generated code (e.g., `clientFrameworks: "vue"` → Vue SFC instead of React+Tailwind; `clientLanguages: "html,css"` → plain HTML/CSS)? Hypothesis: per the schema wording they are logging-only and do **not** alter output — but this must be tested, because if they *do* steer output it changes the whole read methodology.

**Tools + exact params (same node each call):**
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId, clientFrameworks, clientLanguages }`, varying the two override fields.

**Procedure** (on `repr-card`):
1. Baseline: `get_design_context` with `clientFrameworks: "react"`, `clientLanguages: "typescript"`. Record the framework/language of the returned code.
2. `clientFrameworks: "vue"`, `clientLanguages: "javascript"`. Record output framework.
3. `clientFrameworks: "django"`, `clientLanguages: "python,html"`. Record output.
4. `clientFrameworks: "unknown"`, `clientLanguages: "unknown"`. Record output.
5. `clientFrameworks: "swiftui"`, `clientLanguages: "swift"`. Record output.
6. Diff the **code** across all five. Does the output language/framework track the override, or is it always React+Tailwind regardless?

**Confirm / refute signal.**
- **Honored** (steers output): the returned code's framework/language **changes** to match the override in ≥1 case (e.g., Vue input → Vue output).
- **Logging-only** (refutes steering): output is React+Tailwind in **all** cases regardless of the override values — consistent with the schema's "logging purposes" wording.
- Record verbatim the framework actually emitted per case.

**Result:** _____  (date / build: _____)

---

## E09 — Are component **instance OVERRIDE values** returned by the read tools?

**Question / hypothesis.** Open question: when a component **instance** overrides its main component's text, props, visibility, or styles, do the read tools return the **override values** (what the instance actually shows) or the **main-component defaults**? Hypothesis: `get_design_context` reflects override text/props (it renders the instance), but token/variable read-back via `get_variable_defs` may report main-component bindings; and Code Connect output may drop overrides (ties to E05c).

**Tools + exact params (on an instance with known overrides):**
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId }`.
- `mcp__plugin_figma_figma__get_variable_defs` `{ fileKey, nodeId }`.
- `mcp__plugin_figma_figma__get_code_connect_map` `{ fileKey, nodeId }`.
- Ground truth via `use_figma` read-back of `node.overrides` / `componentProperties` / overridden text.

**Procedure.**
1. In a throwaway draft, create a main component (text = "DEFAULT", a bound color variable, a boolean prop). Create an **instance** and override: text → "OVERRIDDEN", swap the bound color, toggle the prop. Capture the instance `nodeId`.
2. **Ground truth.** Via `use_figma`, read and return the instance's `componentProperties`, `overrides`, and the overridden text node's `characters` + `boundVariables`. This is the reference.
3. `get_design_context` on the instance. Does the returned code contain "OVERRIDDEN" (override) or "DEFAULT" (main default)? Does it reflect the swapped color and toggled prop?
4. `get_variable_defs` on the instance. Are the **instance's** bound variables returned, or the main component's?
5. `get_code_connect_map` on the instance. Are override values present (links to E05c)?
6. Tabulate per tool: override-reflected vs default-shown.

**Confirm / refute signal.**
- Per tool: **returns overrides** iff the tool's output matches the step-2 ground-truth override values; **returns defaults** iff it shows the main-component values.
- Headline verdict = does `get_design_context` (the primary design-to-code tool) reflect instance overrides? Plus the per-tool breakdown.

**Result:** dc=_____ | varDefs=_____ | codeConnect=_____  (date / build: _____)

---

## E10 — Does a failed `use_figma` call truly leave zero changes (atomicity)?

**Question / hypothesis.** The grounding claim: `use_figma` is **atomic** — a script that throws makes zero changes, so retry is safe. Open question: is this true for a script that **partially completes** (creates 3 nodes, then throws on the 4th op)? Hypothesis: atomic = on throw, all 3 prior creations roll back; node graph unchanged.

**Tools + exact params.**
- `mcp__plugin_figma_figma__use_figma` `{ fileKey, code, description }` with code that intentionally throws after N successful ops.
- `mcp__plugin_figma_figma__get_metadata` `{ fileKey, nodeId: <parent> }` to count children before/after.

**Procedure** (throwaway draft):
1. `get_metadata` the target parent; record child count `C0`.
2. `use_figma` with code that: creates 3 rects appended to the parent, then on the 4th line does `throw new Error('boom')` (or references an undefined symbol). Record the returned error.
3. `get_metadata` the parent again; record child count `C1`.
4. Compare: `C1 === C0` (atomic — rollback) vs `C1 === C0 + 3` (non-atomic — partial commit).
5. Repeat with a Plugin-API-level failure (e.g., set an invalid font on the 4th op so the throw comes from the API, not user code) — atomicity may differ for API errors vs thrown JS.

**Confirm / refute signal.**
- **Atomic confirmed** iff `C1 === C0` in both the thrown-JS and API-error variants.
- **Refuted** iff any variant leaves partial nodes (`C1 > C0`) — which would make blind retry unsafe and is a critical reliability-skill input.

**Result:** thrownJS=_____ | apiError=_____  (date / build: _____)

---

## E11 — `get_design_context` `disableCodeConnect`: does turning Code Connect off change the output?

**Question / hypothesis.** The schema exposes `disableCodeConnect` ("Only set this when the user directly requests to disable Code Connect"). Open question: on a node that **has** Code Connect mappings, does `disableCodeConnect: true` swap the Code-Connected snippet for freshly-generated code — and is the Code-Connected path materially different (and cheaper/more correct)? This bounds how much Code Connect actually changes the read output, and provides the controlled comparison E05 needs.

**Tools + exact params.**
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId }` (Code Connect on, default) vs `{ fileKey, nodeId, disableCodeConnect: true }`.

**Procedure** (on a Code-Connect-mapped fixture, e.g. the mapped Button from E05):
1. `get_design_context` default (Code Connect active). Record: does the output reference the codebase component (import path / component name from the mapping)? Record token count.
2. `get_design_context` with `disableCodeConnect: true`. Record: is the output now generic generated markup (no codebase import)? Record token count.
3. Diff the two. Confirm the default path actually uses the mapping (codebase import present) and the disabled path does not.

**Confirm / refute signal.**
- **Honored** iff default output contains the mapped codebase reference and `disableCodeConnect:true` output does not. Record the token delta (Code Connect path is often shorter — quantify).
- **Refuted / inert** iff the two outputs are identical (mapping not being applied even by default, or the flag ignored).

**Result:** _____ (token delta: _____)  (date / build: _____)

---

## E12 — `excludeScreenshot` and screenshot-cost isolation (methodology calibration)

**Question / hypothesis.** `get_design_context` includes a screenshot by default; `excludeScreenshot: true` drops it. Open question: how many tokens does the embedded screenshot add to a `get_design_context` call, and is `excludeScreenshot:true` + a separate `get_screenshot` (URL) the cheaper read path overall? This calibrates the metadata-first loop in the README's Method section.

**Tools + exact params (same node):**
- `mcp__plugin_figma_figma__get_design_context` `{ fileKey, nodeId }` vs `{ fileKey, nodeId, excludeScreenshot: true }`.
- `mcp__plugin_figma_figma__get_screenshot` `{ fileKey, nodeId, maxDimension: 1024 }` (URL).

**Procedure** (on `repr-card`):
1. `get_design_context` default → token count `T_full`.
2. `get_design_context` `excludeScreenshot: true` → token count `T_code`. Confirm the code body is otherwise identical.
3. Screenshot share = `T_full − T_code`.
4. Compare `T_code + (get_screenshot URL cost from E07)` vs `T_full`: which read strategy is cheaper when a screenshot is needed anyway?

**Confirm / refute signal.**
- Output = screenshot token share of `get_design_context` + the cheaper-path verdict (inline-screenshot vs `excludeScreenshot` + separate URL screenshot). Measurement, marked `DONE` when filled. Feeds the reliability skill's read-loop recommendation.

**Result:** screenshot share = _____ tokens · cheaper path = _____  (date / build: _____)

---

## Coverage map (task → experiment)

| Open question (from synthesis `expertiseGaps` / `topIssues`) | Experiment |
|---|---|
| `forceCode` exposed + honored; what it toggles | **E01** |
| True single-call `use_figma` node/payload ceiling beyond 20kb/≤10-ops conventions (binary search) | **E02** |
| Does `setReactionsAsync` execute under `use_figma` | **E03** |
| Noise / Texture / Glass effect params settable | **E04** |
| Feb-2026 Code Connect bugs (Component-Set snippet drop, base-variant tokens, missing instance overrides) still live | **E05** |
| Remote `get_metadata` instruction-only / first-design-only pathology | **E06** |
| Per-call token cost head-to-head (metadata vs design_context vs screenshot URL vs base64) on one component | **E07** |
| Does `get_design_context` honor `clientFrameworks` / `clientLanguages` overrides | **E08** |
| Whether instance OVERRIDE values are returned | **E09** |
| `use_figma` atomicity on partial completion | **E10** |
| `disableCodeConnect` effect on output (controlled CC comparison) | **E11** |
| `excludeScreenshot` screenshot-cost isolation (read-loop calibration) | **E12** |

*Verified tool surface introspected live 2026-06-17. Param names (`forceCode`, `excludeScreenshot`, `disableCodeConnect`, `clientFrameworks`, `clientLanguages`, `enableBase64Response`, `maxDimension`, `code.maxLength=50000`) are quoted from the live Claude Code plugin schemas. Confidence on the param surface: high. Confidence on the hypotheses: open — that is what these experiments resolve.*
