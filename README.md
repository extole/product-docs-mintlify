# Extole docs on Mintlify (bake-off canary)

A generated Mintlify port of Extole's documentation, standing up the **Phase 1
Mintlify bake-off** from [`extole/ai-tools#346`](https://github.com/extole/ai-tools/pull/346).

- **Sources:** guides from [`extole/product-docs`](https://github.com/extole/product-docs)
  + OpenAPI bundles from [`extole/extole-specification`](https://github.com/extole/extole-specification).
- **Generator:** [`scripts/convert_from_product_docs.py`](scripts/convert_from_product_docs.py)
  (deterministic, re-runnable).
- **How it was built + how hard it was:** see [`MIGRATION.md`](MIGRATION.md).

## Local preview

```bash
npx mint@latest dev        # http://localhost:3000
npx mint@latest validate   # strict build check (0 errors / 0 warnings)
```

## Regenerate

```bash
python scripts/convert_from_product_docs.py \
  --product-docs ../product-docs \
  --specification ../extole-specification \
  --out .
```

## Deployment

Mintlify's GitHub App deploys the default branch to production and builds a
**preview deployment for every PR**. This canary was landed as a PR so it could
be previewed without touching `main`.
