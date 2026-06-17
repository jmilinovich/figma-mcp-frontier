# QA report — accuracy gate

*Run 2026-06-17. Scope: the seven authored files below, cross-checked against the repo
ground truth (`README.md` "Verified grounding facts", `results/ground-truth-probes.md`,
`results/token-cost.md`, `results/raw/*`) and against the **live tool schemas** for all
19 `mcp__plugin_figma_figma__*` tools (introspected this session via ToolSearch).*

Files reviewed:

- `docs/capability-map.md`
- `docs/methodology.md`
- `docs/experiment-matrix.md`
- `docs/landscape-comparison.md`
- `harness/README.md`
- `fixtures/README.md`
- `skill-draft/SKILL.md`

---

## What was verified clean (no change needed)

**Server identity & tool count.** All seven files name the same subject — the **remote**
server `mcp.figma.com` via Anthropic's Claude Code plugin (`mcp__plugin_figma_figma__*`),
**19 tools**, verified **2026-06-17**. No drift in identity, count, or date across files.

**Remote/shared split.** capability-map's 10-remote-only / 9-shared split was re-counted
from its own table and matches the README (10 remote-only = use_figma, generate_figma_design,
create_new_file, generate_diagram, upload_assets, add_code_connect_map,
get_code_connect_suggestions, send_code_connect_mappings, get_libraries, whoami; 9 shared).

**Live-schema param facts (all confirmed against the loaded schemas):**

- `get_design_context` — required `nodeId`,`fileKey`; optional `clientFrameworks`,
  `clientLanguages` (logging-only), `disableCodeConnect`, `excludeScreenshot`, `forceCode`.
  `nodeId` pattern `^\d+[:-]\d+$`; `fileKey` rejects `undefined`/`null`. ✓
- `get_screenshot` — `maxDimension` default 1024 / max 65536; `enableBase64Response`
  default false; `contentsOnly` default false. ✓
- `get_metadata` — required `fileKey` only; `nodeId` optional (omit → top-level page list);
  design files only. ✓
- `get_variable_defs` — required `nodeId`,`fileKey`; "requires a concrete node target". ✓
- `download_assets` — `defaultFormat` enum `png/jpg/svg/pdf`; `defaultScale` 0.01–4;
  source images capped at 20. ✓
- `use_figma` — `code` maxLength 50000, `description` maxLength 2000, `skillNames` logging;
  mandates `figma-use`; Inter "Semi Bold"/"Extra Bold" gotcha. ✓
- `create_new_file` — required `fileName`,`planKey`,`editorType` (design/figjam/slides);
  `planKey` pattern `^(team|organization)::\d+$`. ✓
- `generate_diagram` — required `name`,`mermaidSyntax`; `planKey` required-if-authenticated;
  supported Mermaid types graph/flowchart/sequenceDiagram/stateDiagram/stateDiagram-v2/
  gantt/erDiagram; the "not supported" list matches. ✓
- `upload_assets` — `count` 1–5 (default 1), `nodeId` only when count==1, `scaleMode`
  FILL/FIT/TILE (default FILL), `batchCommit` default false; PNG/JPG/GIF/WebP, max 10MB,
  SVG not supported (→ `createNodeFromSvg`). ✓
- `add_code_connect_map` / `send_code_connect_mappings` — `label` enum is exactly the
  **16** values listed in capability-map. ✓
- `search_design_system`, `get_libraries`, `get_figjam`, `get_code_connect_map`,
  `get_code_connect_suggestions`, `get_context_for_code_connect` — params all match. ✓

**driftAndCorrections honored.** The 351,378-token (official worst case) vs 657,311-token
(third-party GLips) figures are correct and never swapped in any file (grep-verified).
Dead tool names `get_code`/`get_image` appear **only** as the renamed-from/old names in
rename context — never used as live tools (grep-verified). Rename date (~Oct 17 2025) is
consistent. Write-to-canvas dated 2026-03-24, not Sept 2025, consistently. Non-existent
tools (`get_make_resources`, `create_design_system_rules`, `list_code_components`,
`get_code_component_info`) correctly flagged as not-real / instruction-blurb-only.

**Open questions** are presented as open (not resolved) in every file: capability-map
"Open questions" section, methodology §8, experiment-matrix E01–E12 with blank results,
skill-draft's "OPEN (expertiseGaps)" callouts and "To be tightened" section.

---

## Corrections made

| # | File | What was wrong | Fix |
|---|------|----------------|-----|
| 1 | `docs/capability-map.md` (Auth & seat gating, line ~71) | The `whoami` `plans[]` return shape (`name`/`seat`/`tier`/`seat_type`/`key`) was tagged **`[schema]`**. The live `whoami` schema takes **no params** and documents only that it identifies the authenticated user — it does **not** describe a return shape. Over-claim of schema verification. | Re-tagged the return-shape claim as **`[behavior]`** (from the synthesis), kept the params-none and the `planKey` **pattern** (`^(team\|organization)::\d+$`) as `[schema]` since those *are* schema-verified (via `create_new_file`/`generate_diagram`). Added an explicit note distinguishing the two. |
| 2 | `docs/capability-map.md` (whoami section, line ~303) | Same over-claim: the `handle`/`email`/`plans[]` return shape **and** the `rate-limits-access.md` resource link were asserted under a trailing **`[schema]`** tag. The schema does not expose either. | Re-tagged return shape as `[behavior]`; replaced the unverifiable "response returns a rate-limits resource link `[schema]`" with the schema's *actual* documented purpose (permission-issue diagnosis), quoted verbatim and tagged `[schema]`; moved the MCP-resources-vs-`get_make_resources` point to `[behavior]`. |
| 3 | `skill-draft/SKILL.md` (step 6, generate_figma_design) | Claimed "the resulting **`imageHash`** can then be reused as a fill via `use_figma`." No `imageHash` return is in the `generate_figma_design` schema or grounding facts — fabricated mechanism. The schema's actual contract is two-phase capture (script + `captureId`, poll every 5s ×10) landing a flat raster node, used as a reference then deleted. | Replaced the `imageHash` claim with the schema-accurate two-phase capture flow and the verified "use as layout reference → rebuild with `use_figma` → delete the raster" workflow. |
| 4 | `skill-draft/SKILL.md` (step 4, output cap) | "Tool output is effectively capped around ~20 KB — exceeding it **truncates/fails** the call." "Truncates" softly contradicts the verified read-side finding that the cap **fails the call outright, not truncates**. | Kept ~20 KB as an explicitly-labeled *convention* (not a measured limit); changed wording to "fails the call"; added that the read-side analog (25,000-token per-tool cap) *fails outright rather than truncating*, pointing to `harness/`. Tightened, not watered down. |

**Total corrections: 4** (3 over-claimed `[schema]`/fabricated-mechanism fixes, 1
truncation-vs-fail consistency fix). All confined to the two files where the claims
diverged from the live schemas; no specificity removed.

---

## Notes (checked, deliberately left as-is)

- **GLips package naming.** `figma-developer-mcp` (capability-map, methodology) vs
  `figma-context-mcp` / `Figma-Context-MCP` (landscape-comparison, skill-draft) are both
  legitimate identifiers for the same GLips project — the **npm package** is
  `figma-developer-mcp`, the **GitHub repo** is `Figma-Context-MCP`. landscape-comparison
  anchors the repo name to its GitHub URL. Not an error; left as-is.
- **Rung-1 fixture: frame vs component.** `results/ground-truth-probes.md` records the
  *actually-probed* node 1:2 as an auto-layout **frame** + text; `fixtures/README.md`
  Rung 1 *specifies* a `createComponent()` **component**. These describe two different
  artifacts (a quick live probe vs the forward-looking ladder spec) and are each internally
  consistent. `ground-truth-probes.md` is outside this QA scope. Left as-is.
- **`generate_figma_design` "in parallel with `use_figma`"** (capability-map) does **not**
  contradict **"never parallelize `use_figma`"** — the former parallelizes *two different
  tools*; the latter forbids concurrent *use_figma↔use_figma* calls. Verified against the
  `use_figma` schema's own guidance. Consistent.
- **`MAX_MCP_OUTPUT_TOKENS` / `anthropic/maxResultSizeChars`** (methodology §5) are
  appropriately confidence-tagged (Medium) and flagged as open. Left as-is.

*Gate verified 2026-06-17 against the live `mcp__plugin_figma_figma__*` schemas.*
