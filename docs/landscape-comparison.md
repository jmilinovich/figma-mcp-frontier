# Landscape comparison — the official remote Figma MCP vs the alternatives

*Verified 2026-06-17. Subject under test: the **remote** Figma MCP (`mcp.figma.com`) delivered via Anthropic's official Claude Code plugin (`mcp__plugin_figma_figma__*`), **19 tools**, introspected live on 2026-06-17. Everything else here is sourced and dated inline.*

The point of this file is not to crown a winner. It is to draw the boundary precisely — what the **official** server uniquely does, where a **third-party** server is the right tool, and where you should leave the MCP layer entirely and reach for a **commercial design-to-code** product. Every claim is tagged with its source and date. Vendor self-tests are labeled as such and should not be read as independent benchmarks.

> **Three "Figma MCPs" people conflate.** (1) The **official local** Dev Mode server (`127.0.0.1:3845`, read-only, runs inside the desktop app, beta'd 2025-06-04). (2) The **official remote** server (`mcp.figma.com`, hosted, read **+ write**, beta'd 2025-09-23; `use_figma` write + `generate_figma_design` shipped 2026-03-24) — **this is the subject under test**. (3) **Third-party** servers (Framelink/GLips, cursor-talk-to-figma) — unaffiliated open-source projects. When a blog post says "the Figma MCP does X," check which of the three it means; most don't say. (Source: repo README grounding facts, verified 2026-06-17.)

---

## The contenders

| Server | Maintainer | Type | What it is |
|---|---|---|---|
| **Official remote** (`mcp.figma.com`) | Figma (first-party) | Hosted remote MCP | 19 tools. Read (React+Tailwind code-gen, screenshots, variables, Code Connect), bidirectional **write to canvas** via `use_figma`, design generation. The subject of this repo. |
| **Framelink / GLips** `figma-context-mcp` | Graham Lipsman (GLips) + community | Self-hosted local MCP | Read-only. Returns *descriptive* simplified JSON (layout/style facts) and lets your agent write the code. ~15.1k★. (Source: [github.com/GLips/Figma-Context-MCP](https://github.com/GLips/Figma-Context-MCP), 2026-06-17.) |
| **cursor-talk-to-figma-mcp** | Sonny Lazuardi (+ Grab fork) | Self-hosted local MCP + Figma plugin + WebSocket bridge | Read **and write**. ~40+ granular write tools (`create_rectangle`, `set_fill_color`, `move_node`, `set_instance_overrides`, …). Drives the live desktop canvas through a plugin. ~6.8k★. (Source: [github.com/sonnylazuardi/cursor-talk-to-figma-mcp](https://github.com/sonnylazuardi/cursor-talk-to-figma-mcp), 2026-06-17.) |
| **Commercial design-to-code** (Builder.io Visual Copilot, Anima, Locofy) | Vendors | SaaS plugin / API, *not* MCP | Figma → production component code. Not an agent tool surface; a product you run a design *through*. (Sources below, 2026.) |

---

## Comparison matrix

Scored on the six axes that actually decide which one you pick. "Official" = the remote server under test.

| Axis | Official remote | Framelink/GLips | cursor-talk-to-figma | Commercial (Anima/Builder/Locofy) |
|---|---|---|---|---|
| **Read fidelity** | Highest semantic grounding: returns React+Tailwind **plus** `get_variable_defs`, `get_code_connect_map` (real component → code mapping), `search_design_system`. Knows your design system. | Descriptive JSON (geometry + style names), no codegen, no Code Connect, no design-system awareness. Agent infers structure. | Node-level read (`get_document_info`, node inspection); raw-ish, no Code Connect grounding. | Highest *visual* fidelity to the static design; Anima respects Figma Variables → CSS vars / Tailwind config. No agent-loop read. |
| **Write capability** | **Bidirectional.** `use_figma` runs arbitrary JS against the Figma Plugin API (atomic per call); `generate_figma_design` generates layouts; `create_new_file`, `upload_assets`. Code→design round-trip. | **None** — read-only. (Source: GLips README, 2026-06-17.) | **Write-first**, ~40+ granular mutation tools via the desktop plugin. Most fine-grained write surface of any option. | One-directional (design→code). Builder.io adds a visual CMS to edit *content* post-build, not the Figma file. |
| **Token cost** | **Heaviest.** `get_design_context` routinely exceeds Claude Code's 25,000-token per-tool cap; documented worst case **351,378 tokens** (hard call failure, not truncation). Metadata-first loop is mandatory. (Source: repo grounding facts, 2026-06-17.) | **Leaner read by design** — self-reports **~25% smaller** payloads than official; explicit caveat (verbatim): *"Based on a few admittedly haphazard tests. I should probably do more."* Treat as a **vendor self-test**, not a benchmark. (Source: [framelink.ai](https://www.framelink.ai/), 2026-06-17.) | Per-node calls are small individually, but a full build is *many* round-trips → chatty. No published token numbers. | N/A to the agent context — work happens in the vendor's pipeline, not your context window. |
| **Setup friction** | **Lowest.** Hosted; OAuth via the Claude Code plugin. No local process, no token to manage, no plugin to keep open. | Medium: `npx`, supply a **Figma PAT**, wire into the agent. No desktop app needed. (Source: GLips README, 2026-06-17.) | **Highest:** run the MCP server **and** install/open the Figma desktop plugin **and** keep the WebSocket bridge alive. Desktop app must be open. (Source: Lazuardi README, 2026-06-17.) | Low-medium: install plugin / sign in; API tier for automation. |
| **Headless / PAT support** | **No.** OAuth-gated hosted service tied to an interactive session; not a drop-in for CI with a PAT. | **Yes.** PAT + `npx` = runs headless in CI. Its strongest differentiator. (Source: GLips README, 2026-06-17.) | **No.** Requires the live desktop app + plugin open; cannot run headless. (Source: Lazuardi README, 2026-06-17.) | **Partial → yes (Anima).** Anima exposes API/SDK export for CI/CD pipelines; Builder/Locofy are primarily interactive. (Source: [sixtythirtyten](https://www.sixtythirtyten.co/blog/from-figma-to-code-ai-design-to-dev-workflows-in-2026), 2026.) |
| **Determinism** | **Write is non-deterministic** — `use_figma` is an LLM emitting Plugin-API JS; same prompt ≠ same result. Atomic (failed script = zero changes, safe retry) but not repeatable without a reliability harness. Read is stable. (Source: repo grounding facts, 2026-06-17.) | **Read is deterministic** (it's an API transform, not generation). No write to be non-deterministic. | Writes are explicit tool calls, so individual ops are deterministic; the *agent's plan* over them is not. No reliability guarantees in docs. (Source: Lazuardi README, 2026-06-17.) | **Most deterministic output** — same design in → same code out (vendor-tuned generator, not a free-running LLM). |

> **Myth to retire:** the widely-cited **657,311-token** Figma-MCP blowup is the **third-party GLips** server, **not** the official one. The official server's documented worst case is **351,378 tokens** (`get_design_context` on a heavy node). Don't attribute one's numbers to the other. (Source: repo grounding facts, 2026-06-17.)

---

## When the official remote server wins

Reach for the official server when **grounding in the real design system and round-tripping** matter more than token economy.

- **Code Connect grounding.** `get_code_connect_map` / `get_context_for_code_connect` map Figma components to your *actual* code components. No third-party server has this. If your team has invested in Code Connect, the official server is the only one that pays it back. (Confidence: high — first-party feature, verified in the live tool surface 2026-06-17.)
- **Bidirectional write / code→design.** Only the official remote server and cursor-talk-to-figma can write to Figma at all, and only the official one ships first-party `use_figma` + `generate_figma_design` + `create_new_file` from a hosted endpoint with no plugin to babysit. If you want "turn this React page into a Figma file," this is the path. (Confidence: high.)
- **Round-trip loops** (read design → generate code → push refinements back → re-read). The shared read+write surface plus design-system awareness makes the official server the only single-tool round-trip story.
- **Lowest setup friction / no secrets.** Hosted + OAuth means nothing to install, no PAT to leak, nothing to keep open. Best for an interactive Claude Code session on a real project.
- **Design-system fidelity on read.** `get_variable_defs` + `search_design_system` give the agent the tokens and components by name, not just geometry — the difference between "build it with our `Button`" and "reconstruct a button from rectangles."

## When to reach for a third-party server instead

The official server's two structural weaknesses are **no headless/PAT path** and **heavy read payloads**. When either dominates, go third-party.

- **Headless / CI via PAT → Framelink/GLips.** If the job is "in a build pipeline, fetch design facts with a token, no human and no browser," the official server can't do it (OAuth, interactive). Framelink runs on `npx` + a PAT and is read-only by design — exactly the safe, scriptable shape CI wants. (Source: GLips README, 2026-06-17. Confidence: high.)
- **Leaner read output / avoid context blowups → Framelink/GLips.** When `get_design_context` keeps blowing the 25k cap and you only need layout + style facts (not generated React), Framelink's descriptive JSON is lighter — it self-reports ~25% smaller, **but that is a vendor self-test with the maintainer's own caveat** ("admittedly haphazard tests"); measure it on *your* files before relying on it. (Source: [framelink.ai](https://www.framelink.ai/), 2026-06-17.)
- **Granular, deterministic write control → cursor-talk-to-figma.** When you want to drive precise canvas mutations (create this node, set that fill, override this instance) rather than hand an LLM a whole-script `use_figma` call, the ~40+ explicit tools give per-operation control. The cost is the heaviest setup (desktop app + plugin + WebSocket bridge, must stay open) and no headless story. (Source: Lazuardi README, 2026-06-17. Confidence: high.)

## When to skip the MCP layer entirely

If the goal is **production-quality component code from a finished design**, and you don't need an agent in the loop, a commercial design-to-code product will usually beat any MCP server on output quality and determinism. High-level positioning (sources 2026):

- **Locofy** — cleanest component structure / closest-to-human output; best value for pure codegen.
- **Anima** — strongest for React teams needing token-aware output that respects Figma Variables; **has API/SDK + CI/CD export** (the one commercial option that automates), though its raw output is the most literal/flat. IBM invested Feb 2026.
- **Builder.io Visual Copilot** — widest framework support (React, Vue, Svelte, Angular, Qwik, Solid) **plus a visual CMS** for non-dev content editing post-build.

(Sources: [sixtythirtyten 2026](https://www.sixtythirtyten.co/blog/from-figma-to-code-ai-design-to-dev-workflows-in-2026), [sitegrade 2026](https://sitegrade.io/en/blog/locofy-vs-builder-io-vs-anima-design-to-code-2026/). These are third-party comparison write-ups, not controlled benchmarks — treat positioning as directional.)

---

## Decision shortcut

| If your priority is… | Use |
|---|---|
| Code Connect grounding / design-system-aware read | **Official remote** |
| Code→design write or round-trip from one tool | **Official remote** |
| Zero setup, no secrets, interactive session | **Official remote** |
| Headless / CI with a Figma PAT | **Framelink/GLips** |
| Leanest read payload (verify the % yourself) | **Framelink/GLips** |
| Fine-grained, per-operation deterministic writes | **cursor-talk-to-figma** |
| Production component code, no agent in the loop | **Anima / Builder.io / Locofy** |

---

## Open questions (to resolve empirically, not assert)

- **Is Framelink's ~25% lighter read real on representative files?** The only number is the maintainer's self-described "haphazard" self-test. This repo's harness should measure official `get_design_context` vs Framelink `get_figma_data` on the same fixture ladder and publish the delta. **Open.**
- **Does descriptive JSON (Framelink) or React+Tailwind (official) yield better downstream code** when the codebase has an established component kit? Plausibly task-dependent; not yet measured here. **Open.**
- **Per-build token cost of cursor-talk-to-figma's chatty node-level model** vs one heavy official `use_figma` script — no published numbers; needs a controlled build. **Open.**

---

## Sources

- Official remote server facts — figma-mcp-frontier repo README "Verified grounding facts," verified 2026-06-17 (live introspection of `mcp__plugin_figma_figma__*`, 19 tools).
- [github.com/GLips/Figma-Context-MCP](https://github.com/GLips/Figma-Context-MCP) and [its README](https://github.com/GLips/Figma-Context-MCP/blob/main/README.md) — read-only, descriptive JSON, PAT/`npx`, ~15.1k★ (2026-06-17).
- [framelink.ai](https://www.framelink.ai/) — "~25% smaller" / "24% smaller" payload claim and the verbatim caveat "*Based on a few admittedly haphazard tests. I should probably do more.*"; "works with all agents" (2026-06-17). **Vendor self-test.**
- [github.com/sonnylazuardi/cursor-talk-to-figma-mcp](https://github.com/sonnylazuardi/cursor-talk-to-figma-mcp) — read+write, ~40+ tools, MCP server + Figma plugin + WebSocket bridge, desktop app required, ~6.8k★ (2026-06-17).
- [sixtythirtyten — Figma-to-Code 2026](https://www.sixtythirtyten.co/blog/from-figma-to-code-ai-design-to-dev-workflows-in-2026) and [sitegrade — Locofy vs Builder.io vs Anima 2026](https://sitegrade.io/en/blog/locofy-vs-builder-io-vs-anima-design-to-code-2026/) — commercial design-to-code positioning (third-party write-ups, not benchmarks).
