# fixtures — the controlled corpus spec

*Subject under test: the **remote** Figma MCP (`mcp.figma.com`) via Anthropic's Claude Code plugin (`mcp__plugin_figma_figma__*`), 19 tools. Verified **2026-06-17**.*

This directory specifies the **inputs** the benchmark harness measures against. It does not contain Figma files (you can't check a Figma file into git); it contains the **reproducible recipe** for the two corpora, precise enough that an independent tester on their own Figma account regenerates the same fixtures and gets comparable numbers.

## Why two corpora

The two measurement axes of this repo pull in opposite directions, so we use one corpus for each.

| Axis | What it needs | Corpus |
|------|---------------|--------|
| **Write success-rate & token cost of `use_figma`** | Inputs *we* author, with known-correct ground truth, escalating in complexity along one controlled dimension at a time | **(A) Complexity Ladder** — built by us, in throwaway drafts |
| **Read fidelity of `get_design_context` / `get_variable_defs` / `get_metadata`** | A real-world design file that is (a) public so anyone can repro, (b) backed by published code so we have an objective answer key | **(B) shadcn/ui** — public Figma community kit ↔ public shadcn codebase |

- The **ladder** is a *controlled* instrument. Because we author every rung and increase exactly one kind of complexity per rung, the resulting `use_figma` write-success and token curves are attributable: if rung 3 (variables) fails where rung 2 (variants) succeeded, the variable-binding step is the culprit, not noise. It deliberately exercises the write frontier the README flags as fragile — variant matrices, variable/token bindings, atomicity under non-determinism.
- **shadcn** is a *naturalistic* instrument with a built-in **answer key**. The shadcn codebase is the canonical, public source-of-truth for what a "Button" or "Card" *is* in code; the shadcn Figma community kit is a widely-used design rendering of those same components. Reading the kit and diffing against the code tells us how much real design intent survives the `get_design_context` round-trip — and because both sides are public, any reader can reproduce the exact diff. This is the read-fidelity signal the field argues about with anecdotes.

Net: the ladder gives **controlled write curves**; shadcn gives **reproducible real-world read fidelity**. Neither alone is enough.

---

## Corpus A — the Controlled Complexity Ladder

**Built by us** with `mcp__plugin_figma_figma__use_figma` into a **throwaway draft file** (per the repo method: drafts have no Full-seat gate and zero blast radius; never build into a shared/library file). Each rung adds **exactly one** axis of complexity over the previous rung. Build rungs **sequentially, one `use_figma` call sequence at a time** — never parallelize `use_figma` (parallel calls corrupt library builds, per the README).

### Setup (once)

1. Create a fresh draft: load the `/figma-create-new-file` skill, then `create_new_file` (editorType `figma`). One file per ladder run; name it `frontier-ladder-<YYYY-MM-DD>`.
2. Confirm identity/seat with `whoami` and record it in the result file — write behavior is seat-dependent (`use_figma` is gated to **Full seats** for non-draft files; drafts are exempt, which is why we use drafts).
3. Record the live tool build/date (`get_metadata` against any node, or the plugin version banner) so the run is version-stamped.

### The rungs

Each rung's **intent** below is the spec. The harness translates intent → a single `use_figma` script (JS against the Figma Plugin API) and records: pass/fail, atomic-rollback-or-not, retries to green, total tokens, and a `get_design_context` read-back diff vs the stated intent.

#### Rung 1 — single Button (baseline)

One Button component (not a frame, an actual **component** via `figma.createComponent()`). Establishes the floor: can `use_figma` create one well-formed, named component at all, and what does it cost.

- **Geometry:** auto-layout, horizontal, padding 12×24 (V×H), corner radius 8, hug contents.
- **Fill:** solid hex (raw value, *no* variable binding yet — that's rung 3).
- **Text:** child text node "Button", a Google Font only (e.g. Inter Medium 14 — `use_figma` has **no custom-font support**, Google Fonts only). Load the font before setting characters.
- **Naming:** component named exactly `Button`.
- **Ground truth:** 1 component, 1 text child, auto-layout on, hug sizing. Read-back via `get_design_context` should yield a single button with the right padding/radius/label.
- **Controlled variable introduced:** none (baseline).

#### Rung 2 — Button with a full variant matrix

The **same** Button, now a **component set** (`figma.combineAsVariants()`) spanning a full 3-property matrix. This isolates the *variant-authoring* cost/failure mode.

- **Properties (cartesian product):**
  - `size`: `sm` | `md` | `lg`
  - `variant`: `primary` | `secondary` | `ghost`
  - `state`: `default` | `hover` | `disabled`
- **Count:** 3 × 3 × 3 = **27 variants** in one component set, each with correct variant property values on the name (`size=md, variant=primary, state=hover`).
- **Per-variant fidelity:** sizes change padding/text size; `variant` changes fill/stroke (primary = filled, secondary = outline, ghost = transparent); `state` changes opacity/fill (disabled = reduced opacity, hover = darker fill). Still **raw hex** — no variables yet.
- **Ground truth:** 1 component set, 27 children, all three variant properties present and correctly enumerated; read-back should surface the variant axes (the README flags `generate_figma_design` as *unable* to emit variant props — this rung tests whether `use_figma` can, and whether the read tools expose them).
- **Controlled variable introduced:** variant matrix only (geometry/tokens unchanged from rung 1 in kind).

#### Rung 3 — composed Card using real variables/tokens

A **Card** component composed from the rung-2 Button, now wired to **Figma Variables** (the design-token primitive `get_variable_defs` reads). This isolates the *variable-binding* axis — the README's flagged-fragile step.

- **Variables (create a collection `tokens` first, then bind — never inline hex on bound properties):**
  - Color: `color/bg/surface`, `color/bg/muted`, `color/fg/default`, `color/fg/muted`, `color/border/default`, `color/accent` (bind these — do not hardcode).
  - Radius: `radius/sm` (4), `radius/md` (8), `radius/lg` (12).
  - Space: `space/2` (8), `space/3` (12), `space/4` (16).
- **Composition:** Card = auto-layout vertical frame bound to `color/bg/surface` fill, `radius/lg` corner, `space/4` padding, `color/border/default` stroke; containing a title text (`color/fg/default`), a body text (`color/fg/muted`), and **an instance of the rung-2 Button** (not a copy — a real instance) set to `variant=primary, size=md`.
- **Ground truth:** every color/radius/space property reads back as a **bound variable alias**, not a raw value. `get_variable_defs` should return the full `tokens` collection; `get_design_context` on the Card should show the Button as an *instance*. This rung is where token-binding write-success is expected to dip — that dip is the measurement.
- **Controlled variable introduced:** variable/token bindings + component composition (instancing).

#### Rung 4 — multi-section screen

A full screen assembled from the lower rungs. This isolates *scale/assembly* — the regime where reads blow the 25,000-token per-tool cap (README worst case **351,378 tokens**, a hard call failure) and where write atomicity matters most.

- **Layout:** a desktop frame (1440 wide) with ≥4 distinct sections: a top **nav/app bar** (logo + nav items + a `Button ghost`), a **hero** section (heading + subcopy + primary/secondary Buttons), a **content grid** of ≥6 Card instances (rung 3), and a **footer**. All auto-layout, all colors/spacing bound to the `tokens` collection.
- **Reuse:** sections must use **instances** of the rung-2 Button and rung-3 Card — not redrawn primitives. The screen should contain dozens of instances total.
- **Ground truth:** one screen frame, nested auto-layout, ≥6 Card instances + multiple Button instances, fully token-bound. This is the read-cost stress case: run the metadata-first loop (`get_metadata` → targeted `get_design_context` per section → `get_variable_defs`) and record where/whether a naive whole-screen `get_design_context` exceeds the 25k cap.
- **Controlled variable introduced:** scale + multi-section assembly (no new primitive kind).

### What the ladder measures (per rung)

- **Write success:** did the `use_figma` script apply cleanly (atomic — a failed script makes *zero* changes, so retries are safe)? Retries-to-green.
- **Token cost:** total tokens for the write sequence, and for the read-back loop.
- **Fidelity:** read-back diff vs the stated intent (missing variant props, dropped variable bindings, instances flattened to copies, font substitutions).
- **Determinism:** re-run the same intent into a fresh draft N times; `use_figma` is **non-deterministic**, so we report variance, not a single number.

---

## Corpus B — the shadcn/ui corpus

A **public, reproducible** read-fidelity corpus with a code-side answer key.

### Two sides

- **Code ground-truth:** the public **shadcn/ui** codebase (`ui.shadcn.com` / `github.com/shadcn-ui/ui`). This is the objective spec for each component's structure, variants, and tokens (shadcn components are the canonical Radix-+-Tailwind reference). Pin a commit SHA in the result file so the answer key is version-stamped.
- **Design source:** a **public shadcn/ui Figma community kit** (a community-published shadcn design kit). The kit must be **duplicated into the test account** — `get_design_context` reads files the authenticated account can open; a community file you haven't duplicated may not be reachable. Record the exact kit (publisher + community URL) and the duplicated file key in the result file, because community kits vary in completeness and that variance is part of the finding.

> ⚠️ **Reproducibility caveat (confidence: medium):** there is no single official Figma-authored shadcn kit; multiple community kits exist and they differ. Pin *which* kit by URL + duplicated file key. If a reader uses a different kit, read-fidelity numbers are comparable in *method* but not directly in *magnitude*. This is a known limitation of the naturalistic corpus and is exactly why the ladder (which we fully control) exists alongside it.

### Components to benchmark

Read these five from the kit and diff against their shadcn code definitions:

| Component | Code answer-key focus | Read-fidelity question |
|-----------|----------------------|------------------------|
| **Button** | `variant` (default/secondary/destructive/outline/ghost/link) × `size` (default/sm/lg/icon) | Do variants/sizes survive as variant props, or collapse to flat frames? |
| **Input** | base field, focus/disabled states, border/ring tokens | Do interaction states + ring tokens read back, or only the resting state? |
| **Card** | composed: Header/Title/Description/Content/Footer slots | Does composition/slotting survive, or flatten into one node? |
| **Alert** | `variant` (default/destructive), icon + title + description | Does the destructive variant + icon slot map back to the code shape? |
| **Badge** | `variant` (default/secondary/destructive/outline) | Smallest surface — baseline for "does a trivial component round-trip cleanly?" |

### What the shadcn corpus measures (per component)

- **Structural fidelity:** does `get_design_context` return code whose component shape (slots, children, variant axes) matches the shadcn source? Score missing/extra/renamed parts.
- **Token fidelity:** does `get_variable_defs` surface the kit's design tokens, and do they line up with shadcn's CSS-variable theme (e.g. `--primary`, `--muted-foreground`, `--radius`)?
- **Read cost:** per-component token cost of the metadata-first loop vs a naive direct `get_design_context`.
- **Naming-drift robustness:** confirm we're calling **`get_design_context`** and **`get_screenshot`** (the renamed-from `get_code` / `get_image` tools) — the README flags stale names as the #1 source of breakage; the harness must never reference the dead names.

---

## Notes on method (shared by both corpora)

- **Read loop:** metadata-first — `get_metadata` (sparse XML structural outline) → targeted `get_design_context` (returns **React+Tailwind code**, not XML) on specific nodes → `get_variable_defs` as needed. Avoid whole-file `get_design_context` (the 25k-token cap; worst case 351,378 tokens = hard failure, *not* truncation).
- **Write loop:** `use_figma` only, sequential, into drafts. It runs **arbitrary JS against the Figma Plugin API**, is **atomic** (failed script = zero changes, safe retry) and **non-deterministic** (report variance). It **cannot fetch external image URLs** and supports **Google Fonts only** — so fixtures must never depend on remote images or custom fonts.
- **Out of scope here:** `generate_figma_design` is *not* a ladder builder — it emits **flat dead raster** (no instances, no variant props, no variable bindings), so it can't produce the token-bound, instanced fixtures the ladder requires. It may be benchmarked separately as a write-frontier baseline, but it is not how Corpus A is built.
- **Version-stamping:** every result references the build date, the tool + exact params, `whoami` seat, the ladder draft file key, and (for shadcn) the pinned code SHA + duplicated kit file key.

---

*Point-in-time against a fast-moving target. Verified **2026-06-17**.*
