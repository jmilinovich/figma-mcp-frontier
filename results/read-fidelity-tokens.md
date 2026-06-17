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

## Open follow-ups
- Multi-mode (light/dark) `get_variable_defs` — does it collapse to default mode? (needs a 2-mode collection)
- Does Code Connect (`get_code_connect_map`) further upgrade this to real component imports rather than a raw `<div>`? (needs a mapped component)
