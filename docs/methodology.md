# Methodology

**How the figma-mcp-frontier benchmark is run, and why its numbers should be trusted.**

*Verified build: the **remote** Figma MCP (`mcp.figma.com`) via Anthropic's official Claude Code plugin (`mcp__plugin_figma_figma__*`), **19 tools**, introspected live **2026-06-17**.*

> Every measurement in this repo is **point-in-time against a fast-moving target.** Figma ships to this server continuously, renames tools out from under its own docs (`get_code` → `get_design_context`, `get_image` → `get_screenshot`), and is moving `use_figma` from free beta toward usage-based paid. A number here is a claim about *one build on one date*, not a permanent property of "the Figma MCP." Re-run before you cite.

---

## 0. Scope: what is under test, and what is not

This benchmark targets the **official, first-party, remote** server only.

- **Under test:** the hosted server at `mcp.figma.com`, reached through the Claude Code plugin namespace `mcp__plugin_figma_figma__*` — the 19 tools enumerated in [`capability-map.md`](./capability-map.md).
- **Explicitly NOT under test, and never conflated with it:**
  - the **local** desktop Dev Mode server (`127.0.0.1:3845`, read-only, 9 shared tools) — a different surface with a different transport and a different token ceiling;
  - **third-party** servers — Framelink/GLips `figma-developer-mcp` and `cursor-talk-to-figma`. The widely-quoted **657,311-token** blowup is the *GLips* server. This server's verified worst case is **351,378 tokens** (`get_design_context` on a heavy node). Do not cross-cite the two.

When a result could plausibly be attributed to the wrong server, the result line names the transport (`remote`) and the plugin tool id. (Confidence: the remote/local/third-party split is **High** — grounded in the live tool list and the README corrections.)

---

## 1. What is measured

Four axes, each with its own capture path and rubric:

| Axis | Question it answers | Primary tools exercised | Unit |
|---|---|---|---|
| **Read fidelity** | When we read a design into code, how correct and complete is the result? | `get_metadata`, `get_design_context`, `get_variable_defs`, `get_screenshot`, `get_code_connect_map` | 0–3 per dimension, 5 dimensions (see §3) |
| **Per-tool token cost** | How many tokens does each tool/param combination cost on a given node? | all read tools, measured per call | response-size tokens per call |
| **Write success-rate** | When we ask `use_figma` to build a thing, does it run, and does the produced node match intent? | `use_figma`, `create_new_file`, `generate_figma_design`, `upload_assets` | atomic pass/fail + 0–3 fidelity (see §4) |
| **Determinism** | Does the same input give the same output across runs? | any tool run **N** times | variance across N runs |

These are deliberately separated. A tool can be **cheap and wrong** (a sparse `get_metadata` outline), **expensive and right** (`get_design_context` that blows the cap but is faithful within the part it returns), or **right but non-deterministic** (`use_figma` producing a correct-but-different node each run). Collapsing them into one "score" hides exactly the trade-offs this repo exists to expose.

---

## 2. The measurement loop: metadata-first, low-token

The read benchmark mirrors the loop a competent agent should actually use — it is both the recommended usage pattern *and* the measurement protocol:

1. **`get_metadata`** first. This returns a **sparse XML structural outline** (node ids, types, hierarchy) — cheap. It is *not* code. Use it to locate the node id(s) worth reading in full.
2. **Targeted `get_design_context`** on the specific node id(s) found in step 1 — never the whole page. `get_design_context` returns **React + Tailwind code** (not XML), and is the tool that blows the 25k cap (§5). Scoping it to a node is the single biggest token lever.
3. **`get_variable_defs`** only when token/variable resolution is in question, to capture the design-token bindings behind the code.
4. **`get_screenshot`** as the visual ground-truth oracle for scoring read fidelity (the "what it should look like" reference), and `get_code_connect_map` when checking component-reuse fidelity.

Measuring along this loop means the token numbers reflect **how the tool is meant to be driven**, not a naive "dump the whole file" worst case. The naive worst case is *also* recorded separately (it is where the 351,378-token failure lives), but it is labeled as the anti-pattern, not the headline.

---

## 3. Read-fidelity rubric

The output under scoring is the code (and bindings) returned by the read loop in §2, judged against the design as rendered (`get_screenshot`) and against the source file's actual structure. Five dimensions, each **0–3**, scored against explicit anchors. Max **15**.

For each call we also log whether the result was **truncated or cap-failed** (§5); a cap-failed call is scored **0 on Completeness** regardless of the quality of what little returned, because the agent cannot use a result it never received.

### 3.1 Structural accuracy (0–3)
Does the returned code reproduce the node's real layout structure — nesting, auto-layout direction, constraints, ordering?

| Score | Anchor |
|---|---|
| 0 | Structure is wrong or hallucinated; nesting/ordering does not match the source tree. |
| 1 | Top-level structure roughly right; auto-layout/constraints largely lost or guessed. |
| 2 | Structure correct; minor flatten/regroup or one mis-mapped constraint. |
| 3 | Structure matches the `get_metadata` tree and rendered layout, including auto-layout direction and constraints. |

### 3.2 Token / variable binding (0–3)
Are design-token and variable references preserved as bindings, or flattened to raw literals?

| Score | Anchor |
|---|---|
| 0 | All values are raw literals (`#0D99FF`, `16px`); no variable awareness. |
| 1 | A few tokens surface; most are literals; `get_variable_defs` needed and still partial. |
| 2 | Most bound values resolve to named variables/tokens; a minority leak as literals. |
| 3 | Bound values consistently resolve to the correct named variable/token (verified against `get_variable_defs`). |

### 3.3 Component reuse via Code Connect (0–3)
Where a node is an instance of a known component, does the output reference the mapped code component (Code Connect) instead of re-emitting markup?

| Score | Anchor |
|---|---|
| 0 | Known mapped components re-emitted as raw markup; mapping ignored. |
| 1 | Some instances recognized as components but not the Code-Connect-mapped symbol. |
| 2 | Most mapped instances emit the correct component reference; some props missed. |
| 3 | Mapped instances emit the correct Code Connect component **with** correct props (verified against `get_code_connect_map`). |
| n/a | No Code Connect mappings exist for the node — dimension excluded from that node's max. |

### 3.4 Semantic correctness (0–3)
Do element roles, names, and intent survive — buttons read as buttons, inputs as inputs, the right text content in the right place?

| Score | Anchor |
|---|---|
| 0 | Roles/content wrong or invented; text mismatched or missing. |
| 1 | Generic `div` soup; correct text but lost roles/semantics. |
| 2 | Most roles and text correct; a few generic or mislabeled. |
| 3 | Roles, names, and text content all match the source intent. |

### 3.5 Completeness (0–3)
Did everything in the requested scope come back, uncapped?

| Score | Anchor |
|---|---|
| 0 | Call **cap-failed** (§5) or returned essentially nothing usable. |
| 1 | Truncated; a usable fraction returned, remainder lost. |
| 2 | Nearly complete; minor trailing elements dropped. |
| 3 | Full requested scope returned within the cap. |

A node's read-fidelity score is reported as `sum / applicable-max` (e.g. `13/15`, or `11/12` when Code Connect is n/a), **plus** the truncation flag, **never** as a bare percentage. Two independent raters score each node; disagreements >1 on any dimension go to the human-review gate (§7).

---

## 4. Write-success rubric

A write trial is a single `use_figma` script (or `generate_figma_design` / `create_new_file` call) executed against a **throwaway draft file** (§6). Scored in two parts.

### 4.1 Atomicity gate (pass/fail)
`use_figma` runs arbitrary JavaScript against the Figma Plugin API and is **atomic**: a script that throws makes **zero** changes, leaving the file untouched and the call safe to retry. Each trial records:

- **Ran clean:** script executed, returned without error → proceed to fidelity scoring.
- **Failed atomic:** script threw; **verify zero mutation** (the atomicity guarantee) and record the error. A failed-atomic trial scores **0 fidelity** but is *not* counted as file corruption.

This gate is what makes write success-rate a meaningful fraction: `clean runs / total trials`, with the error taxonomy logged for the failures.

### 4.2 Fidelity of the produced node (0–3)
For trials that ran clean, score the resulting node against the stated intent for that fixture:

| Score | Anchor |
|---|---|
| 0 | Ran, but produced node does not match intent (wrong type, wrong content, empty frame). |
| 1 | Recognizable but materially off — major properties wrong, dead raster where structure was asked for. |
| 2 | Matches intent structurally; minor property/spacing/token deviations. |
| 3 | Matches intent including component instances, variant props, and variable bindings where those were requested. |

### 4.3 Known fidelity ceilings (recorded as fixed properties, not per-trial failures)
Two limits are intrinsic to the current build and are noted on every affected trial rather than re-discovered each run:

- **`generate_figma_design` captures flat dead raster** — no component instances, no variant props, no variable bindings. Staff-acknowledged; the #1 write-frontier complaint. Any trial relying on it for *editable* output is capped at **fidelity ≤1** by construction.
- **`use_figma` cannot fetch external image URLs and has no custom-font support (Google Fonts only).** Trials that need either are marked **out-of-envelope**, not failed.

(Confidence on both ceilings: **High** — README-verified, staff-acknowledged.)

---

## 5. How token cost is captured

Token cost is measured as the **response size of each tool call, per tool and per exact parameter set**, on a known fixture node. The capture rules:

- **Per-call, per-params:** the same tool on the same node with different params (e.g. scoped node id vs. whole page) is a *different* measurement. Params are logged verbatim (§6).
- **The 25,000-token per-tool cap.** Claude Code enforces a **25k-token per-tool-call output cap**. `get_design_context` routinely exceeds it. When it does, the call **fails entirely — it is not silently truncated** at the model boundary; the agent gets nothing usable. The verified worst case on this server is **351,378 tokens** for a single `get_design_context` call — recorded as a *total call failure*, not a truncation. This is why §3.5 scores a cap-failed call as 0 completeness.
- **`MAX_MCP_OUTPUT_TOKENS`.** The environment variable that governs the MCP output ceiling on the harness side. The run record stamps its value so a cheap result on a generous limit is never confused with a cheap result on a tight one. Changing it changes what "cap-failed" means, so it is part of the run's identity.
- **The `_meta` `anthropic/maxResultSizeChars` override.** Individual calls can raise the per-result size ceiling via `_meta: { "anthropic/maxResultSizeChars": <n> }`. When a trial sets it, the value is logged on that trial. **Image-tool exception:** `get_screenshot` (image output) does **not** honor `anthropic/maxResultSizeChars` the way the text tools do — image payloads are governed differently — so screenshot trials record raw payload size and explicitly note that the char-override does not apply. (Confidence: **Medium** on the precise image-exception mechanics — flagged below as an open question to nail down empirically.)
- **Tokens vs. chars.** The cap is expressed in tokens (25k) but `maxResultSizeChars` is expressed in characters. The harness logs both the raw char size and the token count for each text response, and records the tokenizer/model used, so the two ceilings can be reconciled per build.

Every token figure carries: tool id, exact params, `MAX_MCP_OUTPUT_TOKENS` in effect, any `_meta` override, char size, token count, and the cap outcome (`ok` / `cap-failed`).

---

## 6. Reproducibility protocol

A result is only trustworthy if someone else can re-derive it. The protocol:

1. **Build-date stamping.** Every result line carries the date it was captured (`verified 2026-06-17`). Because the server moves, an undated number is treated as invalid. The capability map and results are dated files for the same reason.
2. **Exact params logged.** Every call records its tool id and full parameter object verbatim — including node ids, `_meta` overrides, and the `MAX_MCP_OUTPUT_TOKENS` in effect. "We ran `get_design_context`" is not a result; "we ran `get_design_context` on node `1:23` with no `_meta`, `MAX_MCP_OUTPUT_TOKENS=25000`, on 2026-06-17" is.
3. **Draft-files-only for writes.** All write trials (`use_figma`, `generate_figma_design`, `create_new_file`) build into **throwaway draft files**. This (a) avoids the **Full-seat gate** — `use_figma` writes to non-draft files require a **Full** seat; **Dev** seats are read-only outside drafts — so the benchmark runs on any seat, and (b) gives **zero blast radius**: nothing of value can be mutated. `whoami` is run at the top of every session to record the seat type, since it determines what is even attemptable.
4. **Never parallelize `use_figma`.** Write trials run **one `use_figma` call sequence at a time, strictly serial.** Parallel `use_figma` calls **corrupt library builds** — a known failure mode. The harness must not fan out write trials, even across "different" files. (Confidence: **High** — README-verified.)
5. **Fixtures are versioned.** Reads run against the controlled **component-complexity ladder** and the **shadcn corpus** in [`fixtures/`](../fixtures); writes run against fixed build-intents. The fixture spec is committed so the "same input" is genuinely the same.
6. **One transport, named.** Every result names the `remote` transport and the `mcp__plugin_figma_figma__*` tool id, so it can never be silently attributed to the local desktop server or a third-party one (§0).

---

## 7. Handling non-determinism

`use_figma` is **non-deterministic**: the same build-intent can produce a correct-but-different node, or a clean run on one attempt and an atomic failure on the next. Reads are far more stable but not assumed deterministic. So:

- **N runs, not one.** Every write trial — and any read trial that shows variance — is run **N times** (N stated per experiment, default N≥5). Single-run write numbers are never published.
- **Report variance, not just the mean.** Write success-rate is reported as `clean / N` with the spread; fidelity is reported as a distribution across the N runs (e.g. "5/5 ran clean, fidelity 3,3,2,3,2"), never collapsed to a lone average. A tool that is "right on average" but bimodal is a different animal from one that is consistently a 2, and the rubric must show that.
- **Human-review gate.** Automated scoring proposes; a human disposes. Any of these routes a trial to mandatory human review before it enters `results/`:
  - the two raters disagree by >1 on any read-fidelity dimension (§3);
  - a write trial's automated fidelity score and the rendered screenshot visibly disagree;
  - a result would overturn or extend a README correction (those are load-bearing claims and get extra scrutiny);
  - variance across the N runs spans more than one fidelity band.
- **No fabricated resolutions.** Where the protocol has a known unknown, it stays an **open question** (§8) until measured. We do not publish a confident number we have not run.

---

## 8. Open questions (to be resolved empirically — not yet answered)

These are flagged as *open*, with a planned experiment, not as settled results:

- **`get_screenshot` size governance.** Exactly how the image tool's payload ceiling behaves vs. the text tools' `anthropic/maxResultSizeChars`, and whether any override affects it at all. *Experiment:* sweep node sizes and `_meta` values on `get_screenshot`, log raw payload size and outcome. (Currently Medium-confidence; see §5.)
- **`get_design_context` cap behavior at the boundary.** Whether there is any partial/streaming return between "fits" and "351k-token total failure," or whether it is strictly all-or-nothing once over 25k. *Experiment:* binary-search node complexity around the cap.
- **`use_figma` non-determinism sources.** Whether variance is dominated by the model's generated script, the Plugin API runtime, or font/asset resolution — and whether tighter, more deterministic scripting raises the clean-run rate. *Experiment:* hold the script fixed across N runs vs. regenerating it each run, compare variance.
- **Seat-gate edges.** Precisely which `use_figma` operations a **Dev** seat can perform inside drafts vs. what triggers the Full-seat gate. *Experiment:* matrix the write fixtures across recorded seat types via `whoami`.

When one of these is measured, it moves out of this list into [`findings.md`](./findings.md) with a date, the exact params, and a confidence tag — and, if it touches a README correction, through the human-review gate first.

---

## 9. Why these numbers are trustworthy (and where they are not)

**Trust comes from:** one named transport (never conflated with local/third-party); the metadata-first loop matching real usage; explicit 0–3 anchored rubrics scored by two raters with a human gate; token costs carrying full param + cap + `_meta` context; writes confined to zero-blast-radius drafts, run serially, N times, with variance reported; and every line date-stamped to a known build.

**Trust does NOT extend to:** permanence. This is a measurement of one fast-moving build on one date. Tool names, the cap, seat gates, and `use_figma`'s pricing and behavior are all moving. Treat every figure as `verified 2026-06-17` and **re-run before you rely on it.**

---

*Methodology verified against the live remote Figma MCP on **2026-06-17**. Subject: `mcp.figma.com` via the Claude Code plugin (`mcp__plugin_figma_figma__*`), 19 tools. Point-in-time by design.*
