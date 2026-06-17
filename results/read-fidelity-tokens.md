# HEADLINE: does `get_design_context` hardcode tokens, or emit references? — live build 2026-06-17

**The single most-cited complaint about the Figma MCP is that generated code hardcodes hex/spacing instead of using design tokens. This is a controlled test of that claim. The verdict: the complaint is misattributed — the MCP does the right thing when the design is built right.**

## The controlled comparison

Same tool (`get_design_context`), same account, same draft, two fixtures differing only in token binding:

| Fixture | Token binding | `get_design_context` output |
|---|---|---|
| **Rung 1 — Button** | none (raw fills/padding) | **Hardcoded literals:** `bg-[#17171c] px-[16px] py-[10px] rounded-[8px] text-[#fafafa] text-[14px]` |
| **Rung 3 — Card** | fills/stroke/radius/padding/gap/text bound to variables that carry WEB code-syntax | **Token references:** `bg-[var(--color-surface,white)]`, `border-[var(--color-border,#e4e4e7)]`, `gap-[var(--space-lg,24px)]`, `p-[var(--space-lg,24px)]`, `rounded-[var(--radius-lg,12px)]`, `text-[color:var(--color-foreground,#0a0a0b)]`, `text-[color:var(--color-muted,#71717a)]` |

Rung-3 full output:

```jsx
export default function Card() {
  return (
    <div className="… bg-[var(--color-surface,white)] border border-[var(--color-border,#e4e4e7)] border-solid … flex flex-col gap-[var(--space-lg,24px)] … p-[var(--space-lg,24px)] … rounded-[var(--radius-lg,12px)] size-full" data-node-id="3:2" data-name="Card">
      <p className="font-['Inter:Semi_Bold'] font-semibold … text-[16px] text-[color:var(--color-foreground,#0a0a0b)] w-full" data-node-id="3:3">Card title</p>
      <p className="font-['Inter:Regular'] font-normal … text-[14px] text-[color:var(--color-muted,#71717a)] w-full" data-node-id="3:4">Supporting copy bound to a muted token.</p>
    </div>
  );
}
```

## Verdict — RESOLVED (high confidence)

- `get_design_context` **faithfully emits `var(--token, fallback)` references — but ONLY when the design binds variables that have a WEB code-syntax string set.** The code-syntax I set on each variable (e.g. `var(--color-surface)`) surfaced verbatim, with the variable's resolved value as the Tailwind arbitrary-value fallback (`,white`, `,24px`, `,#e4e4e7`).
- The widely-reported "it hardcodes hex/spacing" behavior is therefore **a design-side problem, not a tool limitation**: it happens when values are unbound, or bound to variables that lack code-syntax. Build the design with code-synced variables and the tool does the right thing.
- **Practical implication for the reliability skill:** the highest-leverage thing a write/round-trip pipeline can do for downstream code quality is (1) create variables with explicit scopes AND WEB code-syntax, and (2) bind every fill/stroke/radius/spacing/text-color to them. The token fidelity of the generated code is *bought* at write time.

## Corroborating tool: `get_variable_defs`

`get_variable_defs(node 3:2)` →
```json
{"var(--color-foreground)":"#0a0a0b","var(--color-muted)":"#71717a","var(--space-lg)":"24","var(--radius-lg)":"12","var(--color-surface)":"#ffffff","var(--color-border)":"#e4e4e7"}
```
- Keys are the **WEB code-syntax** strings; values are **resolved** (not aliases). Single-mode here, so the known "default-mode-only / no multi-mode" limitation is not exercised — that's a separate multi-mode test (open).

## Bonus headline: `get_design_context` on a COMPONENT_SET → a parametric typed component

Reading the rung-2 Button **component set** (`node 4:8`, variant property Primary/Secondary/Ghost) did **not** return three separate snippets, nor only the default variant. It returned a single **parametric React component with a TypeScript prop union derived from the variant property**:

```tsx
type ButtonProps = { className?: string; variant?: "Primary" | "Secondary" | "Ghost" };
export default function Button({ className, variant = "Primary" }: ButtonProps) {
  const isGhost = variant === "Ghost";
  const isSecondary = variant === "Secondary";
  return (
    <div className={className || `… rounded-[var(--radius-lg,12px)] ${isGhost ? "" : isSecondary ? "bg-white border border-[#e3e3e8] border-solid" : "bg-[#17171c]"}`} …>
      <p className={`… ${["Secondary","Ghost"].includes(variant) ? "text-[#17171c]" : "text-[#fafafa]"}`}>Button</p>
    </div>
  );
}
```

- **Verdict (high confidence):** raw `get_design_context` handles variant sets **excellently** — it synthesizes the prop type, defaults the variant, and writes per-variant conditional styling. This is the highest-fidelity read behavior observed.
- **Refines a research worry:** the reported "snippets vanish / base-variant-only" problem is specific to the **Code Connect** path (`get_code_connect_map` on a component that became a set), **not** raw `get_design_context`. The two paths must not be conflated. (Code Connect on a set = separate open test.)
- **Internal consistency check passed:** the bound radius surfaced as `var(--radius-lg,12px)`; the *unbound* secondary border stayed hardcoded `#e3e3e8` — exactly the token rule above. Bind it → token; don't → literal.

## Multi-mode `get_variable_defs` — RESOLVED (live 2026-06-17)

Added a second mode ("Dark") to the `tokens` collection with distinct values, then read the Card three ways:

| Node mode | `get_variable_defs(3:2)` returned |
|---|---|
| default ("Mode 1" / light) | `color-surface:#ffffff, color-foreground:#0a0a0b, color-muted:#71717a, color-border:#e4e4e7` |
| explicit **Dark** (`setExplicitVariableModeForCollection`) | `color-surface:#0a0a0d, color-foreground:#fafafa, color-muted:#a1a1ab, color-border:#29292e` |

- **Verdict (high confidence):** `get_variable_defs` **follows the node's active mode** but returns **only that one mode's resolved values** — never the multi-mode definition, never aliases. The "Dark" mode's existence is completely invisible in a light-mode read and vice-versa.
- **Asymmetry confirmed:** you can *write* multi-mode collections fine (Dark mode added cleanly on Pro tier — multi-mode is **not** gated out on Pro), but you cannot *read* the theming through this tool. To capture both modes you must read the node twice under two explicit modes, or drop to the REST API for the full multi-mode/alias definition.
- Refines the research's blunter "default-mode-only" claim → it's **single-resolved-mode, follows the node.**

## Open follow-ups
- Does **Code Connect** (`get_code_connect_map` after `add_code_connect_map`) upgrade output to a real component import — and does the variant-set brittleness reproduce there? (needs a mapping)
- The ~350× gap between measured clean-design cost (Card ~452 tok) and the cited "~162K-token card" — strong evidence those blowups are about **node density / imported cruft / inline rasters**, not semantic complexity. Reproduce on a dense imported design (rung 4 / shadcn corpus).
