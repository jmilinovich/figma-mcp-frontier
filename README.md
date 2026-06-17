# figma-mcp-frontier

**An empirical map of the official first-party Figma MCP server — grounded in the live tool surface, not the docs (which drift badly).**

The field runs on anecdotes. Everyone has a hot take on "the Figma MCP"; nobody has published controlled numbers on read fidelity, per-tool token cost, or `use_figma` write success-rate. This repo is the attempt to fix that: a version-dated capability map, a reproducible benchmark harness, and a frontier demo — all verified against the **remote** Figma MCP as it actually behaves, on a known build, on a known date.

> Subject under test: the **remote** Figma MCP (`mcp.figma.com`) delivered via Anthropic's official Claude Code plugin (`mcp__plugin_figma_figma__*`), **19 tools**, introspected live on **2026-06-17**.

## Why this exists

The official Figma MCP is three launches wearing one name, and almost every public writeup conflates them. Tool names have been renamed out from under the docs. The published tool count is cited as 14, 16, 17, and 21 — it's **19**. "Write to canvas" is routinely dated to Sept 2025; it shipped **2026-03-24**. This repo is the corrected, continuously-verified record.

## Deliverables

> **Latest run:** see [`STATUS.md`](STATUS.md) for what's proven so far and where to pick up.

| # | Artifact | Status |
|---|----------|--------|
| 1 | **Corrected capability map** — all 19 tools, real (often undocumented) parameter behavior, limits, failure modes | ✅ v1 → [`docs/capability-map.md`](docs/capability-map.md) |
| 2 | **Reproducible benchmark harness + live results** — read fidelity, per-tool token cost, write success-rate | 🟢 **8 of 12 experiments resolved** + 4 write rungs → [`results/`](results/) |
| 3 | **Frontier demo** — one thing the docs call aspirational (codebase → tokenized design system, or capture→relink loop) | ⏳ next |
| 4 | **Reliability skill** — make the non-deterministic write pipeline repeatable (atomicity + skill-gating + per-call validation) | 🚧 draft → [`skill-draft/SKILL.md`](skill-draft/SKILL.md) |

### Findings so far (live build 2026-06-17)
- **The "MCP hardcodes tokens" complaint is misattributed** — `get_design_context` emits `var(--token,fallback)` refs iff the design binds code-synced variables (controlled A/B). [`results/read-fidelity-tokens.md`](results/read-fidelity-tokens.md)
- **Component sets read back as one parametric typed component**, not snippets. (same file)
- **`clientFrameworks`/`clientLanguages` are logging-only** — output is always React+Tailwind. [`results/ground-truth-probes.md`](results/ground-truth-probes.md)
- **Per-tool token cost, measured:** `get_metadata` ~98 · `get_screenshot`(URL) ~120 · `get_design_context` ~368 tok (+inline screenshot). [`results/token-cost.md`](results/token-cost.md)
- **3/4 write-ladder rungs built clean, zero retries** (Button, variant set, token-bound Card). [`results/write-success.md`](results/write-success.md)

## Verified grounding facts (the corrections)

*Adversarially verified — 9 of 18 first-pass research claims were wrong and were corrected.*

- **Three launches, one name:** local read-only Dev Mode server (`127.0.0.1:3845`) beta'd **2025-06-04**; remote hosted server (`mcp.figma.com`) beta'd **2025-09-23**; **write-to-canvas (`use_figma`) + `generate_figma_design` shipped 2026-03-24** — *not* Sept 2025.
- **This session is the remote server via the Claude Code plugin** — 19 tools, zero drift vs `developers.figma.com`. 10 are remote-only (all write/generation); 9 are shared with desktop.
- **Status is split, not GA:** read + code-gen are GA; `use_figma` write is **beta, free now, slated to become usage-based paid**, gated to **Full seats** for non-draft files (Dev seats are read-only outside drafts).
- **The flagship read tool was renamed** `get_code` → **`get_design_context`** (~Oct 17 2025) and `get_image` → **`get_screenshot`** — the single biggest source of stale-doc breakage ("failed to execute get_code"). Old names are gone from the live surface.
- **`get_design_context` returns React+Tailwind code, not XML** (XML is `get_metadata`'s sparse structural outline). It routinely blows Claude Code's 25,000-token per-tool cap — documented worst case **351,378 tokens** (a total call failure, not truncation).
- **`use_figma` runs arbitrary JS against the Figma Plugin API**, atomic (a failed script makes zero changes — safe retry), but **non-deterministic**. It cannot fetch external image URLs and has no custom-font support (Google Fonts only).
- **`generate_figma_design` captures flat dead raster** — no component instances, no variant props, no variable bindings. Staff-acknowledged; the #1 write-frontier complaint.
- **Myths:** there is no `get_make_resources` tool (Make = MCP resources primitive); `create_design_system_rules` is not a real tool; the widely-cited 657,311-token blowup is the *third-party* GLips server, not this one (this server's worst case is 351,378).

## Repo structure

```
docs/        capability-map.md · methodology.md · experiment-matrix.md · landscape-comparison.md · findings.md
harness/     the reproducible benchmark runner + how to re-run it
fixtures/    the controlled component complexity ladder + the shadcn corpus spec
results/     dated, raw measurements (token cost, write success, ground-truth repros)
skill-draft/ the reliability skill distilled from what the benchmark finds
```

## Method

Every claim is tagged with the **build date** it was verified on, the **tool + exact params** used, and **confidence**. Read measurements use the metadata-first loop (`get_metadata` → targeted `get_design_context` → `get_variable_defs` as needed). Write measurements build into **throwaway draft files** (no Full-seat gate, zero blast radius), one `use_figma` call sequence at a time — never parallelized (parallel `use_figma` corrupts library builds).

---

*Built and maintained by [@jmilinovich](https://github.com/jmilinovich). Findings are point-in-time against a fast-moving target — every file is dated.*
