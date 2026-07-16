# ReadMe → Mintlify migration (canary)

This repo is the **Phase 1 Mintlify bake-off** called for in
[`ai-tools#346`](https://github.com/extole/ai-tools/pull/346) — a real, buildable
port of Extole's documentation onto Mintlify so we can see how it looks and, more
importantly, **how hard the migration actually is**.

It is generated, not hand-authored: one deterministic script converts the live
sources and can be re-run at any time.

## What was migrated

| Source | → | Mintlify |
|---|---|---|
| `extole/product-docs` `docs/` — 461 ReadMe-flavored `.md` (Guides / Product Docs / Technical Docs) | → | **426 `.mdx` pages** (hidden pages dropped) |
| `extole/extole-specification` `openapi/*.json` — 4 bundles, 446 operations | → | **Native OpenAPI reference** (auto-generated; no per-endpoint stubs) |
| Per-directory `_order.yaml` sidebar ordering | → | `docs.json` `navigation.tabs` → `groups` → `pages` |

The 444 `reference/` stub files in `product-docs` were **not** ported — Mintlify
generates the entire API reference from the OpenAPI files directly, which is the
single biggest structural win over the ReadMe setup.

## How to reproduce

```bash
python scripts/convert_from_product_docs.py \
  --product-docs /path/to/extole/product-docs \
  --specification /path/to/extole/extole-specification \
  --out .
npx mint@latest validate   # strict build check — currently: 0 errors, 0 warnings
```

## The hard part: ReadMe markdown → MDX

Mintlify pages are **MDX** (JSX-strict); ReadMe markdown is lenient and carries a
bag of ReadMe-only widgets. Every one of these had to be handled or the build
fails. This is where the migration cost lives:

| ReadMe construct | Count | Handling |
|---|---|---|
| `[text](doc:slug)` internal links | 286 | Rewritten to real `/tab/group/slug` paths (95% resolved from a slug→path map) |
| `[text](ref:op)` reference links | 4 | Pointed at `/api-reference` |
| Emoji callouts (`> 📘`, `> 🚧`, `> 👍` …) | 67 files | Converted to `<Info>` / `<Warning>` / `<Tip>` / `<Danger>` components |
| `<Image align={…} alt={512}>` widget | many | Converted to plain `<img … />` (JSX attrs stripped) |
| `<Table align={[…]}>` widget | 40 files | Unwrapped; inner HTML/markdown table preserved |
| `<HTMLBlock>{`…`}</HTMLBlock>` widget | 2 | Unwrapped; invalid `style="…"` attrs stripped |
| `<Anchor href=…>text</Anchor>` widget | 7 files | Converted to `[text](href)` |
| `<Glossary>term</Glossary>` widget | — | Unwrapped to plain text |
| Literal `{` / `}` in prose | 98 files | Escaped `\{` (tag-aware — valid JSX `style={{…}}` left intact) |
| Placeholder pseudo-tags (`<REPORT_NAME>`, `<key>`, `<script>` examples) | several | `<` escaped to `&lt;` via an HTML-tag whitelist |
| Raw HTML comments, unclosed `<br>`/`<hr>`/`<img>` | 35 files | Stripped / self-closed |

Frontmatter is normalized (`title` kept, `excerpt` → `description`; ReadMe-only
keys dropped). Images continue to be served from their existing Intercom/ReadMe
CDN URLs — no asset migration was required.

### What the iteration actually looked like

`mint validate` is a genuinely good migration tool: strict, fast, and it names the
file+line+column of every failure. The error count went **62 → 40 → 50 → 0** across
five passes. The jump *up* to 50 was healthy — fixing the first-line parse errors
unmasked deeper ones the parser had been aborting before. The single highest-impact
bug was subtle and self-inflicted (a placeholder-collision in the converter that
swapped inline-code spans for component tags); once fixed, ~50 errors cleared at
once. **Takeaway: the conversion is very automatable, but only with a
build-validate-fix loop — a naive markdown copy would not compile.**

## Known gaps (canary scope, not blockers)

- **42 broken internal links.** These point at pages excluded from the canary
  (ReadMe `hidden: true` pages, or `reference/` stubs now served by the OpenAPI
  generator, e.g. `/delete-token`, `/extole-cli`). Real migration would either
  include those pages or redirect the slugs.
- **Hidden pages dropped.** Anything `hidden: true` in ReadMe is excluded from nav
  and not generated.
- **Callout emoji mapping is best-effort.** Uncommon emoji fall back to `<Note>`.
- **Large spec render is the thing to eyeball.** `management.json` (3.6 MB / 182
  ops) and `management-expert.json` (4.2 MB / 191 ops) are the "do large bundles
  render without `$ref` breakage" exit test from the bake-off — verify these on the
  preview deploy.

## Bake-off exit tests (from `ai-tools#346`)

| Test | Result |
|---|---|
| Do the large bundles render without dropped features / `$ref` breakage? | **Verify on preview** (all 4 specs wired, validate clean) |
| Can an agent land a doc change via PR end-to-end? | **Yes** — this repo *is* that PR; Mintlify builds a preview deployment per PR |
| Is there a real `llms-full.txt`? | Mintlify generates it natively (confirm on deploy) |
| Migration cost from ReadMe | **~1 script + a validate loop.** Structure and specs port cleanly; the widget/MDX cleanup is the real work and is fully scriptable |
