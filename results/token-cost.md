# Per-tool token cost — live build 2026-06-17

The head-to-head nobody has published. All three read tools called on the **same node** (rung-1 Button `1:2`) so the comparison is apples-to-apples. Payload = the text the model ingests; measured as character count of the response payload(s), tokens estimated at ~4 chars/token. Raw responses are under `results/raw/` (gitignored — they contain short-lived asset URLs).

> **Method note:** char-count of the response payload is a reproducible proxy for ingestion cost. It does **not** count inline-image tokens (see the `get_design_context` caveat). Treat token figures as ±10%. A more precise pass would read the MCP client's own usage logs.

## Rung 1 — trivial node (a Button: frame + 1 text child)

| Tool | Params | Payload chars | ~Tokens | Notes |
|---|---|---:|---:|---|
| `get_metadata` | node 1:2 | 395 | ~98 | ~250 chars (≈63%) is the fixed "you MUST call get_design_context" boilerplate; the actual structural XML is ~145 chars |
| `get_screenshot` | node 1:2 (URL mode, default) | 481 | ~120 | JSON (url + w/h + original_w/h) + curl instructions. **Zero base64.** |
| `get_design_context` | node 1:2 (default) | 1,472 | ~368 | **+ 1 inline screenshot by default** → real additional image tokens on top of the 1,472 text chars |
| `get_design_context` | node 1:2, `excludeScreenshot=true` | 1,472 | ~368 | text identical; inline image suppressed (saves the image tokens) |

## Rung 3 — tokenized Card (frame + 2 text children, all values bound to variables)

| Tool | Params | Payload chars | ~Tokens | Notes |
|---|---|---:|---:|---|
| `get_design_context` | node 3:2, `excludeScreenshot=true` | 1,810 | ~452 | token-bound output (`var(--token,fallback)` refs) — see `read-fidelity-tokens.md` |
| `get_variable_defs` | node 3:2 | 199 | ~50 | the 6 bound tokens, keyed by WEB code-syntax, resolved values |

**Scaling so far:** Button (`get_design_context`) ~368 tok → Card ~452 tok. The token-bound Card is only ~23% larger than the unbound Button despite more nodes — the `var(--token,fallback)` refs cost slightly more chars than bare hex, but the structure is still compact at this complexity. The steep growth shows up at screen scale (rung 4), where it approaches the 25k cap.

### What rung 1 already shows

1. **`get_metadata` is the cheapest probe** (~98 tok) and ~3.7× lighter than `get_design_context` text — and that ratio will widen sharply with node complexity, because metadata grows ~linearly with node count while `get_design_context` emits full styled markup per node. This is the empirical basis for the **metadata-first loop**.
2. **For a trivial node, fixed boilerplate dominates both tools** — `get_metadata` is ~63% boilerplate, `get_design_context` ~half "SUPER CRITICAL" conversion instructions. The *marginal* cost of structure is what scales; the ladder (rungs 2–4) will isolate it.
3. **`get_design_context`'s default inline screenshot is an invisible tax.** The 1,472-char text figure understates real cost; `excludeScreenshot=true` is the cheap win under context pressure (text unchanged, image dropped).
4. **`get_screenshot` URL mode is genuinely cheap** (~120 tok) and decouples the pixels from the context window — the agent fetches the PNG via curl only if it needs to look. Base64 mode (not used here) would inline the whole image.

### Open / next

- Rungs 2–4 (variant matrix → tokenized Card → full screen) to chart how each tool scales with complexity and find where `get_design_context` approaches/exceeds the 25,000-token per-tool cap.
- `get_screenshot` base64 vs URL size delta on a real-sized node (rung 4).
- The documented 351,378-token worst case — reproduce on a large page to characterize the failure mode + whether raising `MAX_MCP_OUTPUT_TOKENS` recovers it or just relocates the blowup.

## Scaling curve & the 25k-cap reality (rung 4 — live 2026-06-17)

`get_design_context` (code-only, `excludeScreenshot=true`) across the controlled ladder:

| Fixture | Nodes | Code chars | ~Tokens | chars/node |
|---|---:|---:|---:|---:|
| Button | 2 | ~600 | ~150 | ~300 |
| Card (token-bound) | 3 | ~1,110 | ~280 | ~370 |
| **Dashboard screen (12 stat cards)** | **51** | **10,026** | **~2,507** | **~197** |

- **No downgrade, no cap hit at 51 nodes.** The screen returned full code at ~2.7k tokens (with boilerplate). The silent metadata-downgrade threshold is therefore well above ~2.7k tokens — much nearer the 25k cap.
- **Extrapolated cap threshold: ~509 nodes** of *clean* content (~197 chars/node → ~100k chars ≈ 25k tokens).
- **This settles the ~350× gap.** A clean semantic Card is ~280 tokens; a clean 51-node screen ~2.5k. The cited "~162K-token card" / 351,378-token worst case cannot be clean semantic markup — they are **imported/vector/raster-dense designs**: absolute-positioned cruft, vector path data, deeply-nested redundant wrappers, inline SVG. Per-node cost on imported art runs 10–50× a clean node, so the cap is hit at *far* fewer than 509 nodes for imported designs. **The blowups are a density/cruft problem, not a semantic-complexity problem** — build clean and the read path is cheap. (Definitive cap-failure repro + `forceCode`-override test best done on the dense shadcn import, not synthetic clean nodes.)
- **DRY lever (measured):** the 12 cloned cards emitted as 12 repeated inline `<div>`s (not componentized). Had they been instances of one component, output would be ~1 component def + 12 short usages — roughly a 3–4× reduction at this size, growing with count. Building with **components/instances** (not clones) both shrinks `get_design_context` output and makes it DRY. → reliability-skill input.

## E12 — base64 vs URL screenshot cost (live 2026-06-17)

Measured on the smallest fixture (77×37 Button):

| Screenshot mode | Payload | ~Tokens | Scales with image size? |
|---|---|---:|---|
| URL (default) | ~481 chars (JSON + curl) | ~120 | **No — flat / O(1)** |
| base64 (`enableBase64Response:true`) | URL text **+** a 1,082-byte PNG → ~1,442 base64 chars | ~360+ | **Yes — O(pixel area)** |

- Even for a trivial 77×37 node, base64 is **~3× the URL cost**; the gap explodes for real-sized screenshots (a 1024px-max render is orders of magnitude larger). URL mode is flat ~120 tokens regardless of image dimensions — which is exactly why it's the default and why `get_design_context`'s default inline screenshot (suppressible via `excludeScreenshot`) is the hidden tax at scale. Only force base64 when the agent genuinely can't fetch URLs.

## E01 + E02 — the 25k cap REPRODUCED: downgrade, `forceCode` override, file-spill (live 2026-06-17)

Built a deliberately dense single frame (**724 nodes**, 181 cloned 3-cell rows) and read it. Both behaviors fired:

| Call | Result | Size | What it was |
|---|---|---:|---|
| `get_design_context(18:3)` **default** | exceeded cap → **spilled to file** | 68,302 chars | **METADATA XML** (725 `<frame>`/`<text>` tags, 0 `export default`) — the server **silently downgraded** code→metadata because the code would be too large |
| `get_design_context(18:3, forceCode:true)` | exceeded cap → **spilled to file** | 138,084 chars | **React CODE** (`export default function CapTestDense()`, 726 `data-node-id`) — `forceCode` **overrode** the downgrade and forced code |

**E01 RESOLVED — the demotion is real and `forceCode` is honored (remote).** When the generated code would be too large, default `get_design_context` returns **metadata XML instead of code** (the documented demotion — confirmed at ~724 nodes). **`forceCode:true` overrides it and forces full code** (68k metadata → 138k code). This **refutes the desktop-reported "forceCode is inert"** — on the remote server it works exactly as the schema says. (Note: even the *metadata* form overflowed the client cap here — the demotion alone doesn't guarantee the result fits.)

**E02 RESOLVED — the cap is the CLIENT token cap, and oversized results spill to a file (not lost).** Both responses exceeded Claude Code's per-tool token cap (`MAX_MCP_OUTPUT_TOKENS`, ~25k default). Crucially, the current Claude Code harness **saves the oversized result to `…/tool-results/<tool>-<ts>.txt`** with a jq/subagent recovery path — it does **not** silently lose it. This **refines the research's "total call failure"**: on this harness the result is recoverable from disk (probe with `jq 'type,length'`, extract slices, or hand to a subagent). The practical mitigation stack: scope smaller / `get_metadata`-first / raise `MAX_MCP_OUTPUT_TOKENS` / read the spilled file with jq.

**Threshold:** the demotion/cap engaged at ~724 dense nodes / ~68k+ char output — consistent with the ~509-clean-node extrapolation above (these rows are denser than bare nodes).
