# Ground-truth probes — live build 2026-06-17

Empirical resolution of open questions from the experiment matrix, run against the **remote** Figma MCP (`mcp__plugin_figma_figma__*`) on a Full-seat account, into throwaway draft `PSSvW47Ry1XDSAVZWQcN7t`. Fixture: rung-1 Button (`node 1:2`, a Primary Button = auto-layout frame + Inter Medium text label).

Each probe: question · method (tool + exact params) · result · verdict · confidence.

---

## E10 — Do `clientFrameworks` / `clientLanguages` retarget `get_design_context` output? → **NO (logging-only). RESOLVED.**

- **Method:** `get_design_context(node 1:2, clientFrameworks="swiftui", clientLanguages="swift", excludeScreenshot=true)` vs the same call with `react`/`typescript,css`.
- **Result:** **byte-identical React+Tailwind output** in both cases. The `swiftui` request returned a React `export default function ButtonPrimary()` with `<div className="bg-[#17171c] …">`, and the response *still* instructed: "The generated React+Tailwind code MUST be converted to match the target project's technology stack."
- **Verdict:** The params do **not** change generated code. This is **by design**, not a bug — the live tool schema states both params are "used for logging purposes to understand which frameworks are being used." `get_design_context` **always emits React+Tailwind**; framework/language conversion is offloaded to the calling agent.
- **Why it matters:** The community treats "Compose/SwiftUI requests still return React" as an open bug (forum reports). It is not a defect — it is the documented contract, just buried in a param description. Anyone benchmarking "multi-framework support" is measuring agent-side conversion, not the MCP.
- **Confidence:** High (schema text + live behavior agree).

---

## E01 — Does `forceCode` change `get_design_context` behavior? → **No-op on small nodes; downgrade-override untested. PARTIAL.**

- **Method:** `get_design_context(node 1:2, forceCode=true, excludeScreenshot=true)` vs default.
- **Result:** Identical React+Tailwind code. No difference.
- **Verdict:** Consistent with `forceCode` only mattering when output is large enough to trigger the silent downgrade-to-metadata path (a small node returns code regardless, so `forceCode` is inert here). The desktop-reported "forceCode is ignored" complaint cannot be confirmed or refuted on a small node.
- **Next:** Re-run on rung-4 (a full multi-section screen) large enough to trip the size threshold; compare `forceCode=true` vs default to see whether it forces code over the metadata downgrade. → see write-success.md once rung 4 exists.
- **Confidence:** Med (small-node behavior clear; the load-bearing large-node case is still open).

---

## Confirmed behaviors (live 2026-06-17)

- **`get_design_context` returns React+Tailwind code, NOT XML.** (XML is `get_metadata`.) Confirmed.
- **Hardcoded literals, no token binding, when no variables exist.** Rung-1 output used `bg-[#17171c] px-[16px] py-[10px] rounded-[8px] text-[#fafafa] text-[14px]` — raw values, because the fixture bound no variables. This is the setup to test the #1 community quality complaint ("agents hardcode hex/spacing instead of binding variables") on rung 3 (a *tokenized* Card): does variable-binding in the design surface as token references in the code? **OPEN → rung 3.**
- **`get_metadata` carries an imperative instruction** ("After you call this tool, you MUST call get_design_context …") — confirms the agentic-ergonomics finding that the tool tries to drive the agent's next step. No "first-design-only" pathology observed on a single node; the multi-design-page pathology is a separate test. **PARTIAL.**
- **`get_design_context` bundles an inline screenshot by default**; `excludeScreenshot=true` removes it (confirmed — no image block in the excludeScreenshot runs). Token-relevant: see token-cost.md.
- **`get_screenshot` defaults to a short-lived URL + curl, zero base64** (`enableBase64Response` defaults false). Confirmed.

---

## Code Connect — the whole cluster is Org/Enterprise-gated → **WALL (RESOLVED for this account)**

- **Method:** `add_code_connect_map(node 4:8 → "Button" @ src/components/ui/button.tsx, label React)` and `get_code_connect_map(node 8:2)`.
- **Result:** **both** returned: *"You need a Dev seat on an Organization or Enterprise plan to use Code Connect."* JM Personal is Pro tier / Full seat — which does **not** qualify.
- **Verdict (high confidence):** the **entire Code Connect path — read AND write — is gated to an Org/Enterprise Dev seat.** It is wholly unavailable on Pro/Full. This refines the capability map: `get_code_connect_map` being "shared" with desktop does **not** mean usable — it's seat-gated independent of server.
- **Consequence:** the "does Code Connect upgrade `get_design_context` to a real codebase import?" experiment is **not runnable on this account** — a documented wall (consistent with treating seat/headless limits as walls, not obstacles to defeat). Resolving it needs an Org/Enterprise Dev seat.

## `get_design_context` on an INSTANCE → component composition even without Code Connect

- **Method:** created an instance (`8:2`) of the Primary Button component, then `get_design_context(8:2, excludeScreenshot=true)`.
- **Result:** it returned a **local `Button` component definition** (with `variant?: "Primary"` prop) **and** an `export default function Button1()` that renders `<Button className=… />` — i.e. it factored the instance into a reusable component + a usage site, rather than flattening to a `<div>`.
- **Verdict:** even **without** Code Connect, `get_design_context` is instance-aware and emits component composition. What Code Connect *would* add is replacing that locally-synthesized `Button` with an import of your **real** codebase component — the missing, seat-gated upgrade. Token binding stayed consistent (bound `rounded-[var(--radius-lg,12px)]`; unbound `bg-[#17171c]`).
- **Confidence:** High (live behavior).
