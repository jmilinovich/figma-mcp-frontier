# Read fidelity vs the real shadcn/ui kit — live build 2026-06-17

The headline comparison: `get_design_context` on the **official shadcn/ui Figma Community kit** (`fileKey LlVoZiB6eoanZEV3N1Ppf2`, Components page `4:6598`) scored against the **canonical shadcn/ui source code** as answer-key. This is the "when does the official server's read path actually produce usable code?" test.

## Scorecard (rubric per `docs/methodology.md`, each dimension 0–3)

| Component (node) | Structural | Token binding | Component reuse | Semantic | Completeness | Total |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| **Button** `13:1598` | 3 | 0 | 0 | 1 | 1 | **5/15** |
| **Input** `13:1589` | 3 | 0 | 0 | 0 | 2 | **5/15** |
| **Command** `57:1305` | 3 | 0 | 0 | 1 | 2 | **6/15** |

**Aggregate ≈ 5.3/15 (~36%).** The shape of the result is consistent and decisive: **near-perfect structural + visual fidelity, near-zero token / reuse / semantic fidelity.** The official server produces a faithful *visual scaffold*, not a shadcn *implementation*.

## What's strong (3/3 structural, every time)

The pixel-level layout is faithful and the values map cleanly onto shadcn's scale:
- Button → `bg-[#0f172a] px-[16px] py-[8px] rounded-[6px] text-white` = shadcn's default `bg-primary px-4 py-2 rounded-md text-primary-foreground` (16px=`px-4`, 8px=`py-2`, 6px=`rounded-md`, #0f172a=slate-900=default primary). Visually correct.
- Command → the full palette (search row, "Suggestions"/"Settings" groups, items with icons + ⌘-shortcuts, dividers, drop shadow) reproduced structurally intact.

## What's weak (the gaps that make it a scaffold, not an implementation)

1. **Token binding 0/3 — hardcoded hex, not tokens.** Every color is a literal (`bg-[#0f172a]`, `border-[#cbd5e1]`, `text-[#334155]`). **Root cause confirmed:** this kit binds color via **Figma Styles** (`slate/900`, `slate/400` …), not **Variables with WEB code-syntax**. So `get_design_context` hardcodes the value and appends a separate *"These styles are contained in the design: slate/900: #0F172A, …"* summary — it does **not** inline `var(--slate-900)`. This is the precise mechanism behind the field's universal "it hardcodes everything" complaint: **the most popular community kit doesn't use code-synced variables.** (Contrast `results/read-fidelity-tokens.md`: a kit built with code-synced variables → clean `var(--token,fallback)` refs.)
2. **Component reuse 0/3 — no real imports.** Every component is regenerated as fresh `<div>`s; no `import { Button } from "@/components/ui/button"`. The fix (Code Connect) is **Org/Enterprise-seat-gated** (see `ground-truth-probes.md`) — unavailable on Pro/Full, so this gap is structural for most users.
3. **Semantic 0–1/3 — wrong primitives.** Button is a `<div>` (+ a `state/type` prop *stub*), not a `<button>` with `cva` variants; the real shadcn Button has 6 variants × 4 sizes via `class-variance-authority` — the Figma instance exposed only the default, so the codegen can't recover the variant system. Input's placeholder is a `<p>`, not an `<input>` — **the generated "input" cannot accept input.** No `<label htmlFor>`, no a11y, no focus states.
4. **Icons → raster, not SVG/lucide.** Command's icons came back as `const imgIconCalendar = "https://figma.com/api/mcp/asset/…"` + `<img>`, and even the group dividers were rastered (`imgSectionItems`). Real shadcn uses `lucide-react` (`<Calendar/>`). Imported icon instances flatten to image-asset URLs — a real, lossy semantic gap.
5. **Composed example ≠ atomic component.** The "Input" node (`13:1589`) is shadcn's *newsletter example* (label + field + Subscribe button + helper), not the atomic `<Input>`. `get_design_context` faithfully renders whatever the node *is* — so picking the right node matters; the kit's showcase nodes are examples, not the bare primitives.

## Verdict

On the canonical real-world kit, the official server's read path is a **high-structural-fidelity, low-semantic-fidelity scaffold (~36%)**. The two levers that would lift it — **Code Connect** (real imports) and **code-synced Variables** (token refs) — are respectively **seat-gated** and **absent from the popular kit**. This is the empirical, quantified version of the "strong scaffold, loses on fidelity" thesis, and it tells you exactly *when* the official server pays off: when your design system is built with code-synced variables and Code Connect on an Org/Enterprise seat. Otherwise you get faithful pixels you must re-tokenize, re-semanticize, and re-componentize by hand.

## Side findings (live, this run)

- **`get_design_context` rejects a page/canvas node.** Calling it on the Components canvas `4:6598` → *"You currently have nothing selected. You need to select a layer first."* It requires a **frame/instance** node, not a page. (So the 25k-cap repro needs a single hundreds-of-node *frame* — no single node in this component-organized kit is that dense; each component is ~1–2k tokens. Cap remains extrapolated, not reproduced — see `token-cost.md`.)
- **Transport instability reproduced.** The `command` read failed once with *"MCP server transport dropped mid-call; response was lost"* then succeeded on retry — the documented #2 issue, hit on a larger read. Retry-after-drop works.
- **Styles-summary channel.** Beyond inline classes, `get_design_context` appends a named-style/Effect digest (fonts, `slate/*`, the drop-shadow `Effect(...)`) — useful for re-tokenizing by hand even though the inline code is hardcoded.
