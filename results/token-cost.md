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

### What rung 1 already shows

1. **`get_metadata` is the cheapest probe** (~98 tok) and ~3.7× lighter than `get_design_context` text — and that ratio will widen sharply with node complexity, because metadata grows ~linearly with node count while `get_design_context` emits full styled markup per node. This is the empirical basis for the **metadata-first loop**.
2. **For a trivial node, fixed boilerplate dominates both tools** — `get_metadata` is ~63% boilerplate, `get_design_context` ~half "SUPER CRITICAL" conversion instructions. The *marginal* cost of structure is what scales; the ladder (rungs 2–4) will isolate it.
3. **`get_design_context`'s default inline screenshot is an invisible tax.** The 1,472-char text figure understates real cost; `excludeScreenshot=true` is the cheap win under context pressure (text unchanged, image dropped).
4. **`get_screenshot` URL mode is genuinely cheap** (~120 tok) and decouples the pixels from the context window — the agent fetches the PNG via curl only if it needs to look. Base64 mode (not used here) would inline the whole image.

### Open / next

- Rungs 2–4 (variant matrix → tokenized Card → full screen) to chart how each tool scales with complexity and find where `get_design_context` approaches/exceeds the 25,000-token per-tool cap.
- `get_screenshot` base64 vs URL size delta on a real-sized node (rung 4).
- The documented 351,378-token worst case — reproduce on a large page to characterize the failure mode + whether raising `MAX_MCP_OUTPUT_TOKENS` recovers it or just relocates the blowup.
