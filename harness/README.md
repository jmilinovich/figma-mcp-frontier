# harness/ — reproduce the benchmark end-to-end

**How to re-run the figma-mcp-frontier measurements yourself.** This is the operational
companion to [`docs/experiment-matrix.md`](../docs/experiment-matrix.md) (what to run) and
[`docs/methodology.md`](../docs/methodology.md) (why it's measured this way). This file is
the *how*: prereqs, the exact run sequence, and how raw output lands under
[`results/`](../results/).

> Verified against the **remote** Figma MCP (`mcp.figma.com`) via Anthropic's Claude Code
> plugin (`mcp__plugin_figma_figma__*`), **19 tools**, on **2026-06-17**. Tool names,
> caps, and seat gates drift fast — re-confirm the surface (Step 0) before trusting any
> dated number below.

---

## The one thing to understand first

**There is no standalone CLI you run.** The Figma MCP tools are not HTTP endpoints you can
`curl`. They are MCP tools that only execute inside an **MCP client that drives them** —
in this benchmark, Claude Code with the Figma plugin connected. The "harness" is therefore
a **protocol an agent follows**, plus the fixtures it acts on and the results schema it
writes into. The agent issues the tool calls; the harness defines *which* calls, *in what
order*, and *what to record*.

Concretely, a run is: a human sets up a seat + connection and a fixture file, then drives
(or instructs an agent to drive) the call sequence in `docs/experiment-matrix.md`,
capturing each raw response and its size into a dated file under `results/`.

### Two hard rules (non-negotiable — they are correctness, not style)

1. **Draft-file-only for every write.** All `use_figma` / `generate_figma_design` /
   `create_new_file` measurements run into **throwaway draft files** owned by the test
   account. Drafts need no Full-seat gate and have zero blast radius. **Never** point a
   write experiment at a shared team file.
2. **Never parallelize `use_figma`.** Issue write calls **strictly one sequence at a
   time**, awaiting completion before the next. Parallel `use_figma` calls corrupt
   library builds (interleaved Plugin-API mutations against shared canvas state). This is
   the single most expensive way to ruin a write run. One in-flight `use_figma` at a time,
   always.

Everything else below is in service of running clean, comparable, dated numbers without
violating those two rules.

---

## Prerequisites

### 1. A Figma account with the right seat

| You want to measure | Minimum seat | Notes |
|---|---|---|
| Reads (`get_design_context`, `get_screenshot`, `get_metadata`, `get_variable_defs`, …) | any seat | works on files you can view |
| Writes into a **draft** file (`use_figma`, `generate_figma_design`, `create_new_file`) | any seat | drafts are exempt from the write gate — this is why the whole write benchmark uses drafts |
| Writes into a **non-draft / shared team** file | **Full seat** | Dev seats are read-only outside drafts; this is a hard product gate, not a quota |

For this benchmark you only **need** a Full seat if you deliberately want to reproduce the
*non-draft* write path. The published write numbers are all draft-file numbers — a human
with any seat can reproduce them. **Honesty note:** the seat gate, account creation, and
billing state are human-only steps. An agent cannot provision a seat or accept the beta
terms for `use_figma`.

`use_figma` write is **beta and free as of 2026-06-17**, but is **slated to become
usage-based paid**. If you reproduce later and writes start costing money, that's expected
drift — record the date and the pricing state you observed.

### 2. The MCP connection (the remote server, not the desktop one)

This benchmark is specifically the **remote** server delivered by the **Claude Code Figma
plugin**, exposing tools as `mcp__plugin_figma_figma__*`. Confirm you are on it, not the
local Dev Mode server (`127.0.0.1:3845`, read-only, 9 tools) and not a third-party server
(Framelink/GLips, cursor-talk-to-figma — those have different surfaces and *different*
token-blowup numbers; the 657,311-token figure people cite is GLips, not this server).

To connect:
- Install the Figma plugin in Claude Code and authenticate to Figma (OAuth, browser
  round-trip — **human step**).
- Verify identity and surface before measuring:
  - call `mcp__plugin_figma_figma__whoami` → confirms the authenticated account/seat.
  - enumerate the live tool list → must be **19** `mcp__plugin_figma_figma__*` tools.

### 3. `MAX_MCP_OUTPUT_TOKENS` guidance

Claude Code caps a single MCP tool response at **25,000 tokens** by default. Two facts make
this central to the read benchmark:

- `get_design_context` returns **React + Tailwind code** (not XML), and **routinely
  exceeds 25,000 tokens**. The documented worst case is **351,378 tokens**, which is a
  **total call failure** (the response is rejected wholesale), **not** a truncation.
- The metadata-first loop exists precisely to stay under this cap.

For measurement you may **raise** the cap (e.g. export `MAX_MCP_OUTPUT_TOKENS` higher in
the Claude Code environment) so a large `get_design_context` returns instead of failing —
**but record both numbers**: the raw response size *and* whether it would have failed at
the default 25,000. The interesting result is the cliff, so don't hide it by raising the
cap silently. For the "what does a real agent experience" experiments, run at the
**default 25,000** so failures are real failures.

---

## Step 0 — re-verify the surface (always, before any run)

The whole repo's premise is that the docs drift. So does the live surface. Before
recording numbers:

1. `whoami` → record account + seat.
2. Enumerate tools → record the **count** (expect 19) and the **exact names**. The
   flagship read tool is `get_design_context` (renamed from `get_code` ~2025-10-17) and
   the screenshot tool is `get_screenshot` (renamed from `get_image`). If you still see
   `get_code` / `get_image`, you are on an old build — note it; the dated numbers will not
   be comparable.
3. Write the surface snapshot into the dated results file's header (see schema below).

Confidence tag everything: a number is only as good as the build it was taken on.

---

## The run sequence

The authoritative, per-experiment list lives in
[`docs/experiment-matrix.md`](../docs/experiment-matrix.md). Each row there names: the
experiment family, the fixture, the exact tool + params, the cap setting, and the rubric.
The matrix is the source of truth; this section is the operating loop you wrap around it.

```
for each experiment family in docs/experiment-matrix.md:
  0. verify surface (whoami + tool count)          # Step 0, once per session
  1. load / create the fixture (see fixtures/)      # draft file for writes
  2. run the family's call sequence, in order       # NEVER parallelize use_figma
  3. capture each raw response + its token size
  4. score against the family's rubric
  5. append to results/<family>-<YYYY-MM-DD>.md
```

### Reads — the metadata-first loop

Reads are measured as the loop a competent agent actually uses, **not** a naive single
mega-call:

1. `get_metadata` — sparse structural outline (this is the XML-ish structure; cheap).
2. targeted `get_design_context` on a **specific node** — the React+Tailwind code for just
   what you need. Record token size. Note whether it crosses 25,000.
3. `get_variable_defs` — only when token/variable bindings are needed.
4. `get_screenshot` — for visual ground-truth / fidelity scoring.

Record the per-tool token cost at **each** step and the **total** to satisfy the read
goal, so the loop's efficiency (vs a single blind `get_design_context`) is visible.

### Writes — one sequence at a time, into a draft

1. `create_new_file` (or use a pre-made empty draft) — **draft file only**.
2. the `use_figma` call sequence for the fixture — **strictly serial**. `use_figma` runs
   arbitrary JS against the Figma Plugin API; it is **atomic** (a failed script makes zero
   changes, so retry is safe) but **non-deterministic** (same input can land differently
   run-to-run — which is exactly why write success-rate is a measured quantity, not a
   constant).
3. validate after each call (screenshot / metadata diff) before issuing the next.
4. record: attempts, successes, what landed vs intended, and any Plugin-API error text.

Known write constraints to record against (don't treat as failures of *your* run — they're
properties of the server): `use_figma` **cannot fetch external image URLs**, has **no
custom-font support** (Google Fonts only), and `generate_figma_design` **captures flat
dead raster** (no component instances, variant props, or variable bindings — the #1
write-frontier complaint, staff-acknowledged).

---

## Fixtures

Inputs live in [`fixtures/`](../fixtures/): the controlled **component-complexity ladder**
(reads — increasing node depth/count to find where `get_design_context` crosses the
25,000-token cap) and the **shadcn corpus spec** (writes — a known component set to build
via `use_figma` so write success-rate is comparable run-to-run). Use the fixtures as-is;
if you must change one, version it and note the change in the results header, or the
numbers stop being comparable across dates.

---

## How results are recorded

One **dated file per experiment family** under [`results/`](../results/):

```
results/
  reads-token-cost-2026-06-17.md
  write-success-rate-2026-06-17.md
  ground-truth-repros-2026-06-17.md
results/raw/                 # gitignored — raw dumps before curation
```

Raw, uncurated capture goes in `results/raw/` (gitignored). Only the curated dated file is
committed.

Each dated file carries this header + per-run rows:

```markdown
# <experiment family> — <YYYY-MM-DD>

- server: remote mcp.figma.com via Claude Code Figma plugin (mcp__plugin_figma_figma__*)
- tool count observed: 19
- account/seat: <from whoami>          # e.g. Full seat / Dev seat
- MAX_MCP_OUTPUT_TOKENS: <default 25000 | raised to N>
- fixture: <fixtures/...> @ <version/commit>
- confidence: high | medium | low

| run | tool + params | raw response size (tokens) | crossed 25k cap? | rubric score | notes |
|-----|---------------|----------------------------|------------------|--------------|-------|
| 1   | get_design_context node=… | 12,430 | no | … | … |
```

What every row must capture:

- **Raw response size** in tokens for each tool call (the headline read metric), and
  explicitly **whether it would fail at the default 25,000** even if you raised the cap to
  capture it.
- **Rubric scores** per the family's rubric in `docs/experiment-matrix.md` (e.g. read
  fidelity vs the screenshot; write "landed-as-intended" pass/fail).
- **Write success-rate** as `successes / attempts` with the non-determinism made visible —
  i.e. run the *same* fixture sequence N times and report the distribution, not a single
  pass/fail.

Date every file. A number with no build date is not a result in this repo.

---

## What requires a human vs what an agent can do

| Step | Who |
|---|---|
| Create Figma account, pick/upgrade seat, accept `use_figma` beta terms | **human only** |
| OAuth the Claude Code Figma plugin to the account | **human** (browser round-trip) |
| Decide cap policy (default 25k vs raised) for a family | human |
| `whoami` + enumerate tools (Step 0) | agent |
| Drive the read/write call sequences | agent (MCP client) |
| Capture sizes, score rubric, write `results/` files | agent |
| Commit curated dated results | human or agent |

Be honest in the writeup about which numbers came from which seat and cap setting — a write
number from a Full seat on a non-draft file is a *different* experiment from a draft-file
write, even with identical fixtures.

---

*Point-in-time against a fast-moving target. Re-run Step 0 and re-date everything before
trusting any figure here. Verified 2026-06-17.*
