# Capability map — the official remote Figma MCP

**The definitive corrected reference for all 19 tools on the official first-party Figma MCP.**

> **Subject under test:** the **remote** Figma MCP (`mcp.figma.com`) delivered via Anthropic's official Claude Code plugin (`mcp__plugin_figma_figma__*`). **19 tools.** Schemas in this doc are transcribed from the **live tool surface introspected on 2026-06-17** — every parameter below is exact, not remembered.
>
> **Confidence legend:** `[schema]` = verified against the live tool schema on 2026-06-17 (highest). `[behavior]` = documented/observed behavior from the verified research synthesis. `[open]` = an open question flagged in the synthesis' `expertiseGaps`, to be resolved empirically — *not* a settled answer.

This map distinguishes three things people constantly conflate:

- **OFFICIAL remote server** (`mcp.figma.com`) — the subject of this repo. Hosted by Figma, reached here through Anthropic's Claude Code plugin. Has the full 19-tool surface including write/generation.
- **OFFICIAL local desktop server** (`127.0.0.1:3845`, Dev Mode) — read-only, runs inside the Figma desktop app. Shares 9 of these tools; lacks all 10 write/generation tools. Beta'd 2025-06-04.
- **Third-party servers** — Framelink/GLips (`figma-developer-mcp`), `cursor-talk-to-figma`, etc. Not covered here. The widely-cited **657,311-token** blowup is the *GLips* server, **not** this one — this server's documented worst case is **351,378 tokens** (see `get_design_context`).

---

## Summary table

| # | Tool | Server | Category | One-liner |
|---|------|--------|----------|-----------|
| 1 | `get_design_context` | **shared** (remote + desktop) | Read | Design→code: returns React+Tailwind reference code + screenshot + metadata for a node |
| 2 | `get_screenshot` | **shared** | Read | Renders a node to PNG (URL+curl by default; base64 opt-in) |
| 3 | `get_metadata` | **shared** | Read | Sparse XML structural outline (node IDs, types, names, x/y/w/h) — the cheap map |
| 4 | `get_variable_defs` | **shared** | Read | Resolves Figma variables (color/spacing/type tokens) bound under a node |
| 5 | `get_figjam` | **shared** | Read | Generates UI code from a FigJam (`/board/`) node |
| 6 | `download_assets` | **shared** | Read | Exports a node render + extracts original raster source images in its subtree |
| 7 | `use_figma` | **remote-only** | Write | Runs arbitrary JS against the Figma Plugin API — the general-purpose write tool |
| 8 | `generate_figma_design` | **remote-only** | Write | Captures a web page / HTML into an existing Figma file (pixel-perfect raster) |
| 9 | `create_new_file` | **remote-only** | Write | Creates a new blank design / FigJam / Slides file |
| 10 | `generate_diagram` | **remote-only** | Write | Creates a FigJam diagram from Mermaid.js syntax |
| 11 | `upload_assets` | **remote-only** | Write | Uploads raster images (PNG/JPG/GIF/WebP) into a file as fills/frames |
| 12 | `get_code_connect_map` | **shared** | Code Connect | Reads existing node→code-component mappings |
| 13 | `add_code_connect_map` | **remote-only** | Code Connect | Writes a single node→code-component mapping |
| 14 | `get_code_connect_suggestions` | **remote-only** | Code Connect | AI-suggested linking strategy for unmapped components |
| 15 | `get_context_for_code_connect` | **shared** | Code Connect | Structured component metadata (props/variants/descendants) for authoring templates |
| 16 | `send_code_connect_mappings` | **remote-only** | Code Connect | Bulk-saves approved Code Connect mappings |
| 17 | `search_design_system` | **shared** | Design system & misc | Text search across design libraries (components / variables / styles) |
| 18 | `get_libraries` | **remote-only** | Design system & misc | Lists subscribed + available design libraries for a file |
| 19 | `whoami` | **remote-only** | Design system & misc | Returns the authenticated user, plans, seats, and plan keys |

**Server split:** 10 tools are **remote-only** (marked above) — they do not exist on the local desktop Dev Mode server. The remaining 9 are **shared** with desktop. The remote-only set is essentially everything that *writes* or *generates*, plus the org-level read tools (`get_libraries`, `whoami`) and the Code Connect write/suggest path.

> **Note on the read/write count:** "10 remote-only" includes `add_code_connect_map`, `get_code_connect_suggestions`, and `send_code_connect_mappings` (Code Connect writes/suggestions), plus `get_libraries` and `whoami`. The desktop server's surface is read + code-gen only. `[behavior]`

---

## The renames and the non-existent tools (read this first)

The single biggest source of stale-doc breakage is that **tools were renamed out from under the documentation and blog posts**:

| Old name (gone from live surface) | Current name | Renamed ~ | Symptom of using the old name |
|---|---|---|---|
| `get_code` | **`get_design_context`** | Oct 17 2025 | `failed to execute get_code` / tool-not-found |
| `get_image` | **`get_screenshot`** | (same era) | tool-not-found |

The old names **do not exist** on the live surface. If you see them in a tutorial, the tutorial is stale.

**Tools that have never existed on this server** (cited in the wild, fabricated or confused with other surfaces):

- **`get_make_resources`** — not a tool. Figma Make is exposed via the **MCP resources** primitive (`ListMcpResources` / `ReadMcpResource`), not a bespoke tool. (`get_design_context` *does* handle `/make/` URLs with an assumed `nodeId` of `0:1` — see below.)
- **`create_design_system_rules`** — not a real tool.
- **`list_code_components`** — not a real tool on this server.
- **`get_code_component_info`** — not a real tool on this server.

The four names above appear in the MCP server's own instruction blurb as aspirational/legacy capability descriptions, but **none are present in the live 19-tool schema set.** `[schema]` Treat any code or doc that calls them as broken.

---

## Auth & seat gating (applies to every write tool)

`whoami` `[schema]` returns the authenticated user plus a `plans[]` array; each plan carries `name`, `seat` (`Full` / `Dev` / `View`), `tier`, `seat_type`, and a `key` of the form `team::<id>` or `organization::<id>`. **That `key` is the `planKey` that `create_new_file` and `generate_diagram` require.** `[schema]`

Gating model `[behavior]`:

- **Read + code-gen tools** (`get_design_context`, `get_screenshot`, `get_metadata`, `get_variable_defs`, etc.): GA, work on any file you can view.
- **`use_figma` write:** **beta, free now, slated to become usage-based paid.** Gated to **Full seats** for non-draft files. **Dev seats are read-only outside of drafts.** A **draft file (in your own drafts folder) bypasses the Full-seat gate** — which is why the benchmark harness builds into throwaway drafts.
- A `View` seat (the starter-tier seat in a typical `whoami` response) cannot write at all.

---

## Read tools (design → code)

### 1. `get_design_context` — shared (remote + desktop)
**Purpose:** the flagship design→code tool. Returns reference code, a screenshot, and contextual metadata for a node, intended to be *adapted* to the target project (not pasted verbatim).

**Params** `[schema]`:
- **Required:** `nodeId` (pattern `^\d+[:-]\d+$`, e.g. `123:456` or `123-456`), `fileKey` (rejects literal `undefined`/`null`).
- **Optional:** `clientFrameworks`, `clientLanguages` (logging only), `disableCodeConnect` (bool), `excludeScreenshot` (bool), `forceCode` (bool), and behavior implied by `forceCode` for oversized output.

**Documented behavior** `[behavior]`: extract `nodeId` + `fileKey` from a `figma.com/design/:fileKey/:fileName?node-id=1-2` URL. Branch URLs (`/design/:fileKey/branch/:branchKey/...`) → use `branchKey` as the `fileKey`. **Figma Make files** (`/make/...`): use the `makeFileKey`, and **only here** assume `nodeId` = `0:1`. Returns a code string + a JSON map of asset download URLs + (by default) a screenshot.

**Real / undocumented behavior + limits** `[behavior]`:
- Returns **React + Tailwind** code by default — **not XML**. (XML is `get_metadata`'s job; people confuse the two.)
- **Routinely exceeds Claude Code's 25,000-token per-tool-result cap.** Documented worst case: **351,378 tokens** on a single dense node — this is a **total call failure, not truncation** (the result is rejected wholesale). This is the dominant read-side failure mode.
- Mitigation: drive a **metadata-first loop** — `get_metadata` to find the right small node, then `get_design_context` on that node only. `forceCode: true` forces code even when the server would otherwise downgrade to metadata for oversized output; use it deliberately, it can re-trigger the cap blowup.
- `excludeScreenshot: true` saves context but the schema itself warns against it.

**`[open]`** The exact node-size → token-count curve (where the cap starts biting on the complexity ladder) is an open empirical question — quantifying it is deliverable #2 (the harness).

### 2. `get_screenshot` — shared
**Purpose:** render a node (or the current selection) to a PNG.

**Params** `[schema]`:
- **Required:** `nodeId`, `fileKey`.
- **Optional:** `maxDimension` (positive int, **default 1024**, max **65536** — caps the *longer* edge), `enableBase64Response` (**default false**), `contentsOnly` (**default false**).

**Documented behavior** `[behavior]`: works on design (`/design/`), FigJam (`/board/`), and Slides (`/slides/`) files. **Not supported for Make (`/make/`).** Branch URLs → `branchKey` as `fileKey`. Response JSON carries both `width`/`height` (rendered PNG) and `original_width`/`original_height` (natural canvas size before clamping), so a caller can decide whether to re-request larger.

**Real behavior + limits** `[behavior]`:
- **Default path is a short-lived URL + curl instructions**, *not* an inline image — this is deliberately token-cheap and strongly preferred. Set `enableBase64Response: true` **only** when the agent can't fetch URLs (no shell / sandboxed). Base64 inline is expensive.
- `contentsOnly: true` renders the node in isolation (excludes floating/overlapping content like page-parented connectors). Default `false` matches what the user sees on canvas.
- URLs expire — download promptly.

### 3. `get_metadata` — shared
**Purpose:** cheap structural outline of a node or page in **XML** — node IDs, layer types, names, positions, sizes. The map you read *before* spending tokens on `get_design_context`.

**Params** `[schema]`:
- **Required:** `fileKey`.
- **Optional:** `nodeId`, `clientFrameworks`, `clientLanguages`.

**Documented behavior** `[behavior]`: the schema explicitly says *"Always prefer `get_design_context`"* for actual code — `get_metadata` is for overview only. **`nodeId` is optional: omit it to get the list of top-level pages (guid + name)**; pass it (e.g. page id `0:1`) to drill in. **Design files only** — **not** FigJam (`/board/`), **not** Slides (`/slides/`), **not** Make (`/make/`). If a URL lacks `node-id`, omit `nodeId` (do not guess).

**Real behavior + limits** `[behavior]`: returns only structure, no styles/variables/code — by design. This is the linchpin of the token-efficient read loop; it's how you avoid the `get_design_context` cap blowup. Sparse output = low token cost.

### 4. `get_variable_defs` — shared
**Purpose:** resolve the Figma **variables** (reusable values: colors, fonts, sizes, spacings) bound under a node, e.g. `{'icon/default/secondary': '#949494'}`.

**Params** `[schema]`:
- **Required:** `nodeId`, `fileKey`.
- **Optional:** `clientFrameworks`, `clientLanguages`.

**Documented behavior** `[behavior]`: **requires a concrete node target** ("this remote tool requires a concrete node target" — there is no whole-file variable dump via this tool). **Not supported for Make files.** Branch URLs → `branchKey`.

**Real behavior + limits** `[behavior]`: returns *resolved* values bound under the node, not the full variable collection / mode structure. To map a token system you walk it node-by-node. Pairs with `search_design_system` (which can return variables across whole libraries).

### 5. `get_figjam` — shared
**Purpose:** generate UI code from a **FigJam** node.

**Params** `[schema]`:
- **Required:** `nodeId`, `fileKey`.
- **Optional:** `includeImagesOfNodes` (**default true**).

**Documented behavior** `[behavior]`: **FigJam only** (URL path `/board/`). A `/design/...` fileKey is *not* a FigJam file and must not be passed here. Default `nodeId` `0:1` is the root if none provided. Extract from `figma.com/board/:fileKey/:fileName?node-id=1-2`.

**Limits** `[behavior]`: this is the FigJam-side analog of `get_design_context`; less battle-tested than the design path.

### 6. `download_assets` — shared
**Purpose:** export a node's render **and** extract the original uploaded source images found as fills anywhere in its subtree.

**Params** `[schema]`:
- **Required:** `fileKey`, `nodeId`.
- **Optional:** `defaultFormat` (`png` / `jpg` / `svg` / `pdf`), `defaultScale` (number, **0.01–4**).

**Documented behavior** `[behavior]`: returns (1) an exported image of the node and (2) a list of original raster source images (JPEG/PNG/GIF/WebP) in the subtree, **capped at 20**, each tagged with its real `format`. **Export precedence:** passing `defaultFormat`/`defaultScale` overrides the node's configured export settings; omitting them uses the node's settings, else `png`@`1`. Only set them when the user explicitly asks for a format/scale. Works on `/design/`, `/slides/`, `/board/`; **not** `/make/`.

**Real behavior + limits** `[behavior]`: URLs are temporary — download promptly. Source-image extraction is capped at 20 images per call. For cross-file image transfer, feed the raw image URLs into `upload_assets`.

---

## Write tools (code → design)

> All five are **remote-only** — none exist on the local desktop server. All are gated as in **Auth & seat gating** above.

### 7. `use_figma` — remote-only ⭐
**Purpose:** the general-purpose write tool. **Runs arbitrary JavaScript against the Figma Plugin API** (`figma` global) to create/edit/sync designs, variables, styles, components, layouts.

**Params** `[schema]`:
- **Required:** `fileKey`, `code` (JS, **maxLength 50,000 chars**), `description` (**maxLength 2,000**).
- **Optional:** `skillNames` (logging; pass only when skill docs instruct).

**Documented behavior** `[behavior]`: works on design (`/design/`), FigJam (`/board/`), and Slides (`/slides/`). The schema **mandates loading `figma-use` guidance first** (the `/figma-use` skill, or the `skill://figma/figma-use/SKILL.md` MCP resource) — *"skipping this causes common, hard-to-debug failures."* Choose `use_figma` for **all** writes by default; `generate_figma_design` only for first-time web-page capture. Documented gotchas in the schema: for **Inter**, the weight style is `"Semi Bold"` (with a space) not `"SemiBold"`, likewise `"Extra Bold"`; setting `figma.currentPage` is not sufficient on its own (truncated guidance — follow the skill).

**Real behavior + limits + failure modes** `[behavior]`:
- **Atomic:** a script that throws makes **zero** changes — so retries are safe (no half-applied state).
- **Non-deterministic:** the same intent can yield different results across runs; this is the central reliability problem the planned reliability skill (deliverable #4) targets.
- **Cannot fetch external image URLs** from inside the JS sandbox — route images through `upload_assets` instead.
- **No custom-font support** — **Google Fonts only**. Custom/brand fonts fail.
- **Never parallelize `use_figma` calls** against the same build — parallel calls corrupt library builds. Run one call-sequence at a time.
- `code` is hard-capped at 50,000 chars, forcing large builds to be chunked across sequential calls.

**`[open]`** The empirical **write success-rate** of `use_figma` over a full design-system build (and how much the `figma-use` skill + per-call validation moves it) is unmeasured — it is deliverable #2/#4.

### 8. `generate_figma_design` — remote-only
**Purpose:** capture/import a **web page (by URL) or HTML into an *existing* Figma design file** — a pixel-perfect raster snapshot.

**Params** `[schema]`:
- **Required:** `fileKey` (**design files only** — `/design/`; `/slides/`, `/board/`, `/make/` rejected).
- **Optional:** `captureId` (poll handle; **single-use, one page per ID**), `nodeId` (append capture under this node; else a new page is created).

**Documented behavior** `[behavior]`: two-phase. Call with `fileKey` and **no** `captureId` → get a capture script + a `captureId`; then poll with that `captureId` **every 5s, up to 10 times**, until status is `completed`. Requires a pre-existing file — if none, call `create_new_file` first (load the `figma-create-new-file` skill) and reuse the returned key. For **local** projects, identify the dev-server URL from the codebase first; for **external** URLs, capture via Playwright MCP (don't `open` hash-fragment URLs on external sites). Intended to run **in parallel with `use_figma`** for web apps: `generate_figma_design` lays down the pixel-perfect reference, `use_figma` rebuilds it from real design-system components, then you delete the raster reference.

**Real behavior + limits + failure modes** `[behavior]`:
- **Captures flat, dead raster.** No component instances, no variant properties, no variable bindings, no auto-layout semantics — just pixels. **Staff-acknowledged.** This is the **#1 write-frontier complaint**: the output looks right but is structurally inert and not a usable design-system artifact on its own. Hence the "use it as a reference layer, then rebuild with `use_figma` and delete it" workflow.

### 9. `create_new_file` — remote-only
**Purpose:** create a new blank Figma file (design / FigJam / Slides).

**Params** `[schema]`:
- **Required:** `fileName`, `planKey` (pattern `^(team|organization)::\d+$`), `editorType` (enum: `design` / `figjam` / `slides`).
- **Optional:** `projectId` (else the file lands in the authed user's drafts folder).

**Documented behavior** `[behavior]`: the schema **mandates loading the `/figma-create-new-file` skill BEFORE every call** (plan-resolution contract). To get a `planKey`, call `whoami` and use a plan's `key` verbatim; if the user has multiple plans, ask which. `projectId` can be extracted from `figma.com/files/.../project/:projectId` URLs. Returns the new `file_key` + URL.

**Real behavior + limits** `[behavior]`: omitting `projectId` puts the file in **drafts** — which is exactly the **Full-seat-gate bypass** the harness relies on for zero-blast-radius write benchmarking.

### 10. `generate_diagram` — remote-only
**Purpose:** create a FigJam diagram from **Mermaid.js** syntax.

**Params** `[schema]`:
- **Required:** `name`, `mermaidSyntax`.
- **Optional:** `fileKey` (existing FigJam to add to; else creates its own file), `planKey` (`^(team|organization)::\d+$` — **required if authenticated**), `userIntent`, `useArchitectureLayoutCode`.

**Documented behavior** `[behavior]`: supported Mermaid types only — `graph`, `flowchart`, `sequenceDiagram`, `stateDiagram`, `stateDiagram-v2`, `gantt`, `erDiagram`. **Not supported:** class diagrams, timelines, venn diagrams, generic Figma designs, font changes, moving individual shapes. **Do not call `create_new_file` first** — `generate_diagram` makes its own file. **You MUST surface the returned URL to the user as a markdown link.** Syntax constraints baked into the schema: default `LR` direction for graph/flowchart/ER; quote all shape/edge text; no emojis; no literal `\n`; no `"end"` in classNames; no color in gantt; no notes in sequence diagrams.

**Limits** `[behavior]`: it's a one-shot generator — editing/repositioning afterward must happen in the Figma UI (or via `use_figma`), not by re-prompting this tool.

### 11. `upload_assets` — remote-only
**Purpose:** upload raster images into a Figma file (as fills on a node, or as new frames).

**Params** `[schema]`:
- **Required:** `fileKey`.
- **Optional:** `count` (1–5, **default 1**), `nodeId` (sets the asset as a fill on that node; **only when count == 1**), `scaleMode` (`FILL` / `FIT` / `TILE`, **default FILL**), `batchCommit` (**default false**).

**Documented behavior** `[behavior]`: call with a `count` to get that many **single-use upload URLs**; POST raw bytes to each with the correct `Content-Type`. With `nodeId` → sets a fill on that node; without → creates new frames with image fills on the current page. By default each URL auto-commits and auto-places; `batchCommit: true` defers to one `commitUrl` you call exactly once after all uploads.

**Real behavior + limits** `[behavior]`:
- **PNG / JPG / GIF / WebP only. Max 10MB per asset.**
- **SVG is NOT supported here** — for SVG you must use `use_figma` and call `figma.createNodeFromSvg()`.
- This is the sanctioned path to get external images into a file given that `use_figma`'s JS sandbox **can't** fetch external URLs (tools #7/#11 are designed to be used together).

---

## Code Connect tools

> Code Connect maps Figma component nodes to real code components so `get_design_context` returns *your* component instead of generic markup. Reads are shared with desktop; the suggest/write path is remote-only.

### 12. `get_code_connect_map` — shared
**Purpose:** read existing node→code mappings.

**Params** `[schema]`: **Required:** `nodeId`, `fileKey`. **Optional:** `codeConnectLabel` (disambiguate when multiple language/framework mappings exist).

**Behavior** `[behavior]`: returns `{ [nodeId]: { codeConnectSrc, codeConnectName } }`. Read-only; safe to call on any viewable file.

### 13. `add_code_connect_map` — remote-only
**Purpose:** write a **single** node→code-component mapping.

**Params** `[schema]`:
- **Required:** `nodeId`, `fileKey`, `source` (path/URL in codebase), `componentName`, `label`.
- **Optional:** `template` (executable JS template → creates a richer **figmadoc** record instead of a simple `component_browser` mapping), `templateDataJson` (template metadata; defaults `{}`), `clientFrameworks`, `clientLanguages`.
- **`label` enum** (16 values): `React`, `Web Components`, `Vue`, `Svelte`, `Storybook`, `Javascript`, `Swift`, `Swift UIKit`, `Objective-C UIKit`, `SwiftUI`, `Compose`, `Java`, `Kotlin`, `Android XML Layout`, `Flutter`, `Markdown`.

**Behavior** `[behavior]`: single-mapping writer. For bulk, use `send_code_connect_mappings`.

### 14. `get_code_connect_suggestions` — remote-only
**Purpose:** get an AI-suggested linking strategy for unmapped components under a node.

**Params** `[schema]`: **Required:** `nodeId`, `fileKey`. **Optional:** `excludeMappingPrompt` (return a lightweight list of unmapped components only, dropping the prompt text + images), `clientFrameworks`, `clientLanguages`.

**Behavior** `[behavior]`: step 1 of the suggest→review→save workflow. Output (especially with the prompt text + images) can be large; `excludeMappingPrompt: true` is the token-cheap variant.

### 15. `get_context_for_code_connect` — shared
**Purpose:** structured component metadata for authoring Code Connect **template** files — property definitions (with types + variant options) and a descendant tree (instances + text nodes with their property references).

**Params** `[schema]`: **Required:** `nodeId`, `fileKey`. **Optional:** `clientFrameworks`, `clientLanguages`.

**Behavior** `[behavior]`: this is the introspection tool you call before writing a `template` for `add_code_connect_map` / `send_code_connect_mappings`. Returns variant/prop structure, not code.

### 16. `send_code_connect_mappings` — remote-only
**Purpose:** **bulk**-save approved Code Connect mappings.

**Params** `[schema]`:
- **Required:** `nodeId`, `fileKey`, `mappings[]`.
- Each `mappings[]` item — **Required:** `nodeId`, `componentName`, `source`, `label` (same 16-value enum as #13). **Optional per item:** `template`, `templateDataJson`.
- **Optional top-level:** `clientFrameworks`, `clientLanguages`.

**Behavior** `[behavior]`: step 3 of suggest→review→save (after `get_code_connect_suggestions`, with human review in between). Batch analog of `add_code_connect_map`.

---

## Design system & misc tools

### 17. `search_design_system` — shared
**Purpose:** text search across design libraries for components, variables, and styles.

**Params** `[schema]`:
- **Required:** `query`, `fileKey`.
- **Optional:** `includeComponents` (**default true**), `includeVariables` (**default true**), `includeStyles` (**default true**), `includeLibraryKeys[]` (restrict to specific libraries; keys come from `get_libraries` or prior search results), `disableCodeConnect`.

**Behavior + limits** `[behavior]`: **results are stable within a turn** — the schema says do **not** re-issue identical arguments, *including when the result was empty* (re-querying won't change it that turn). This is the right entry point when `use_figma` needs to build a screen from real library components rather than inventing them.

### 18. `get_libraries` — remote-only
**Purpose:** list the design libraries for a file.

**Params** `[schema]`: **Required:** `fileKey`. **Optional:** `offset` (pagination for org libraries).

**Behavior** `[behavior]`: returns two lists — **subscribed** libraries (added to the file) and **available-to-add** libraries (community UI kits + org libraries). Each entry has `name`, library `key`, `description`, source type. Org-library portion is paginated via `libraries_available_to_add_next_offset` → feed back as `offset`. Use the returned keys to scope `search_design_system` via `includeLibraryKeys`.

### 19. `whoami` — remote-only
**Purpose:** identity + entitlement check; the source of truth for auth/seat gating.

**Params** `[schema]`: none.

**Behavior** `[behavior]`: returns `handle`, `email`, and `plans[]`, each plan carrying `name`, `seat` (`Full` / `Dev` / `View`), `tier`, `seat_type`, and `key` (the `team::<id>` / `organization::<id>` **planKey** consumed by `create_new_file` and `generate_diagram`). Call it first to resolve a `planKey`, and to diagnose permission errors on other tools (confirm the right user is logged in and which seat they hold). The response also returns a `rate-limits-access.md` MCP resource link — corroborating that **MCP resources** (not a `get_make_resources` tool) are how this server exposes docs/Make context. `[schema]`

---

## Open questions (to resolve empirically — do not treat as settled)

- **`[open]` `get_design_context` token curve:** the relationship between node complexity and result token count — specifically where it crosses Claude Code's 25,000-token cap on a controlled complexity ladder. (Deliverable #2.)
- **`[open]` `use_figma` write success-rate:** the measured success-rate of a full design-system build, and how much the `figma-use` skill + atomic retries + per-call validation improve it. (Deliverables #2 / #4.)
- **`[open]` Read fidelity:** how faithfully the `get_metadata` → `get_design_context` → `get_variable_defs` loop reconstructs a known component vs. ground truth, across the complexity ladder. (Deliverable #2.)

---

*Verified against the live remote Figma MCP surface on **2026-06-17** via Anthropic's Claude Code plugin (`mcp__plugin_figma_figma__*`). Parameter facts tagged `[schema]` are transcribed directly from the live tool schemas introspected that day; behavior tagged `[behavior]` is from the verified research synthesis; `[open]` items are unresolved and flagged as such. Point-in-time against a fast-moving target.*
