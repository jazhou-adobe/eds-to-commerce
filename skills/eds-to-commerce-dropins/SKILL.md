---
name: eds-to-commerce-dropins
description: >-
  Convert a vanilla Adobe Experience Manager Edge Delivery Services project
  (the aem-boilerplate) into a Commerce-drop-in storefront based on
  hlxsites/aem-boilerplate-commerce. Use this whenever someone wants to add
  Adobe Commerce / commerce drop-ins (PDP, PLP, cart, mini-cart, checkout,
  account, auth, wishlist) to an existing EDS site, "make my boilerplate a
  storefront", "upgrade aem-boilerplate to commerce", wire up Catalog Service /
  Live Search drop-ins, or migrate an EDS project to the commerce boilerplate —
  even if they don't say "drop-in" by name. There is no official upgrade path,
  so follow this procedure.
---

# Converting an EDS project to Commerce drop-ins

The vanilla [aem-boilerplate](https://github.com/adobe/aem-boilerplate) and the
[aem-boilerplate-commerce](https://github.com/hlxsites/aem-boilerplate-commerce)
are separate templates with no `git` ancestry between them, so there is no
"upgrade" command. Converting means copying the commerce boilerplate's
infrastructure into the target project, regenerating its build artifacts, and
wiring a Commerce backend. This skill is the procedure, distilled from a real,
validated conversion.

**Mental model that prevents most mistakes:** the commerce boilerplate is a
vanilla EDS project *plus* three things — (1) a set of `@dropins/*` npm packages
whose built assets are **generated** into `scripts/__dropins__/`, (2) an
**importmap** in `head.html` that resolves `@dropins/<pkg>/` to those generated
files, and (3) a `scripts/commerce.js` config/fetch layer that the drop-in
initializers and blocks consume. Everything else is ordinary EDS blocks/styles.

## Before you start

- Confirm the target is a **clean, unmodified `aem-boilerplate`**. If it has
  custom blocks/scripts, you must merge rather than replace — diff each file
  instead of copying wholesale, and preserve the custom code.
- You need a **Commerce backend** (Adobe Commerce as a Cloud Service / Catalog
  Service + Live Search endpoints). Without it the storefront installs and loads
  but renders no products. A sandbox/demo tenant is fine for validation.
- Work on a feature branch. The conversion touches `package.json`, `head.html`,
  `scripts/`, `styles/`, and `blocks/` — easy to review as a single diff.
- **Pin to a known commerce-boilerplate release** rather than its moving `main`
  if you want reproducibility (this skill was validated against package version
  **9.0.0**; drop-ins `storefront-pdp@3.2.0`, `storefront-cart@3.3.0`,
  `storefront-wishlist@3.3.0`, `tools@1.9.0`).

## Procedure

### 1. Clone the source of truth

```bash
git clone --depth 1 https://github.com/hlxsites/aem-boilerplate-commerce.git /tmp/abc-src
```

Treat `/tmp/abc-src` as the reference for every copy step. Re-clone if it
disappears — never reconstruct these files from memory.

### 2. Dependencies + build machinery

Copy `.npmrc`, `build.mjs`, `postinstall.js` from the source. Then merge
`package.json`: take the source's `scripts`, `dependencies`, `devDependencies`,
and `overrides`; **keep the target's own** `name`, `private`, `repository`,
`author`, `license` (and decide intentionally about `version`/`description` —
they are not protected, and blindly copying makes the target claim the upstream
version number).

The merged `scripts` must include `install:dropins` (`node build.mjs && node
postinstall.js`), `postinstall`, `postupdate`, and the `build:json*` targets.
The 15 runtime deps are `@adobe/adobe-client-data-layer`,
`@adobe/magento-storefront-event-collector`,
`@adobe/magento-storefront-events-sdk`, the eleven `@dropins/storefront-*`
packages, and `@dropins/tools`.

```bash
mkdir -p scripts/acdl        # see Gotcha 1 — do this BEFORE npm install
npm install                  # runs postinstall → generates drop-in assets
```

`npm install` should end with `✅ Drop-ins installed successfully!`.

### 3. Script wiring

Copy these from the source verbatim (they are EDS-version-matched, so take the
source's `aem.js` too):

```
scripts/scripts.js  scripts/aem.js  scripts/delayed.js
scripts/commerce.js  scripts/ue.js  scripts/ue-utils.js
scripts/initializers/   (one initializer per drop-in)
scripts/components/      (e.g. commerce-mini-pdp)
```

`scripts.js` now imports `loadCommerceEager/loadCommerceLazy/initializeCommerce`
from `commerce.js`; `commerce.js` reads store config and exposes
`CORE_FETCH_GRAPHQL` (core Commerce GraphQL) and `CS_FETCH_GRAPHQL` (Catalog
Service / Live Search). The initializers register each drop-in on the shared
event bus.

### 4. head.html (module resolution — the linchpin)

Replace `head.html` with the source's. It contains the **importmap** mapping
`@dropins/<pkg>/` → `/scripts/__dropins__/<pkg>/`, the `commerce.js` module
script, modulepreload hints, and an importmap shim for older browsers. If you
skip or mangle this, every drop-in import fails with
`Failed to resolve module specifier "@dropins/..."` — that error means head.html
is wrong, not that the packages are missing.

### 5. Blocks, styles, icons, models

```bash
cp -R /tmp/abc-src/blocks/.  blocks/      # ~44 blocks incl. commerce-* + product-*
cp /tmp/abc-src/styles/{styles,lazy-styles,fonts}.css styles/
cp -R /tmp/abc-src/icons/.   icons/
cp -R /tmp/abc-src/models    models/
npm run build:json                        # generates the 3 component-*.json files
```

The commerce `header`/`footer` (mini-cart + search wired) replace the vanilla
ones. Keep any custom blocks the target already had.

### 6. Lint config + .hlxignore

Copy `.eslintrc.js`, `.eslintignore`, `.stylelintrc.json`, `.hlxignore`, and
`418.html`. The eslint config excludes `scripts/__dropins__` (generated code is
not linted). Run `npm run lint` — it must exit 0.

### 7. Backend config

Drop-ins read store config at runtime via
`@dropins/tools/lib/aem/configs.js`. For **local development**, serve it as a
`config.json` at the project root (the dev server exposes it at `/config.json`);
for production it comes from the published `configs` content sheet. The shape:

```json
{
  "public": {
    "default": {
      "commerce-core-endpoint": "https://<tenant>/graphql",
      "commerce-endpoint": "https://<tenant>/graphql",
      "headers": {
        "all": { "Store": "default" },
        "cs": {
          "Magento-Store-Code": "main_website_store",
          "Magento-Store-View-Code": "default",
          "Magento-Website-Code": "base"
        }
      },
      "commerce-assets-enabled": true,
      "analytics": { "base-currency-code": "USD", "...": "..." }
    }
  }
}
```

For Adobe Commerce as a Cloud Service (ACCS) the core and Catalog Service
endpoints are often the same unified GraphQL URL, and no `x-api-key` is needed.
A classic Adobe Commerce + separate Catalog Service tenant needs `x-api-key` and
`Magento-Environment-Id` in the `cs` headers.

### 8. Validate locally

```bash
npx -y @adobe/aem-cli up --no-open --forward-browser-logs   # add --port N if 3000 is taken
```

If the backend has no authored commerce pages yet, validate with **draft pages**
served from a local folder (`--html-folder drafts`). The dev server serves them
under a `/drafts` prefix.

- **PDP** reads its SKU from a `/products/<urlKey>/<sku>` URL (or a `sku` meta).
  Create `drafts/products/<urlKey>/<SKU>.plain.html` containing
  `<div><div class="product-details"></div></div>`.
- **PLP** uses Live Search; create `drafts/search.plain.html` with a
  `product-list-page` block.

In the browser, confirm: the importmap resolves (no module-specifier errors),
`config.json` is fetched, and the drop-ins render. A green run shows the PLP grid
with products/facets and the PDP containers populated.

## File manifest (copy / replace / generate / keep)

| Action | Files |
|---|---|
| **Replace** | `package.json`* · `head.html` · `scripts/scripts.js` · `scripts/aem.js` · `scripts/delayed.js` · `styles/{styles,lazy-styles,fonts}.css` · `.hlxignore` · `.eslintrc.js` · `.eslintignore` · `.stylelintrc.json` |
| **Copy (new)** | `.npmrc` · `build.mjs` · `postinstall.js` · `scripts/commerce.js` · `scripts/ue.js` · `scripts/ue-utils.js` · `scripts/initializers/` · `scripts/components/` · all commerce `blocks/*` · `icons/` · `models/` · `418.html` |
| **Generate** (never hand-copy) | `scripts/__dropins__/**` · `scripts/commerce-events-collector.js` · `scripts/commerce-events-sdk.js` · `scripts/acdl/**` · `component-definition.json` · `component-models.json` · `component-filters.json` |
| **Keep** | the target's custom blocks · `favicon.ico` · existing `fonts/` · LICENSE/README · project `name` in package.json |

\* merge, don't blindly overwrite — preserve identity fields.

## Generated vs. copied (the most common mistake)

The files in the **Generate** row are produced by `npm install` (which runs
`postinstall.js` → copies built drop-in bundles out of `node_modules/@dropins/*`)
and by `npm run build:json` (which merges `models/_*.json`). **Do not** copy them
out of the source clone and **do not** edit them by hand — they'd be stale and
drift from your installed package versions. They *are* committed to git and
*are* served by the CDN (they're the runtime bundles); they're only excluded
from linting. To update drop-ins later: `npm update @dropins/...` then
`npm run postupdate`.

## Gotchas (verified during a real conversion)

1. **Pre-create `scripts/acdl/` before the first `npm install`.** `postinstall.js`
   copies the Adobe Client Data Layer with `fs.copyFileSync`, which does not
   create parent directories — so a first run crashes with `ENOENT` on
   `scripts/acdl/...`. `mkdir -p scripts/acdl` first, or just run `npm install`
   twice.

2. **`.hlxignore` has no explicit `__dropins__` entry.** Generated drop-ins are
   excluded from linting via `.eslintignore`, not `.hlxignore`, because they are
   meant to be served. Don't "fix" this by adding `__dropins__` to `.hlxignore`.

3. **Empty PDP / `$0.00` prices are almost always a backend scoping problem, not
   a code bug.** The drop-ins automatically attach a
   `Magento-Customer-Group` header (for an anonymous shopper this is
   `sha1("0")` = `b6589fc6ab0dc82cf12099d1c2d40ab994e8410c`). If the tenant's
   **Catalog Service / price book is not indexed for that customer group**, the
   `products(skus:)` query returns `{ "data": { "products": [] } }` with HTTP
   200, the PDP emits an empty data event, and the block renders its shell with
   no name/price/image. Diagnose by replaying the failing GraphQL request as
   `curl` **with and without** the `Magento-Customer-Group` header — if removing
   it returns the product, the fix is on the backend (index the catalog/price
   book for that group), not in the conversion. Note Live Search
   (`productSearch`, used by the PLP) is unaffected and still returns results.

4. **Port 3000 may be busy.** `aem up` won't fall back; pass `--port N`.

5. **Console 404s for `placeholders/*.json`, `nav.plain.html`,
   `footer.plain.html`, and `adobe-clean-*.woff2` are expected on a fresh
   backend** — they mean content/fonts aren't authored yet, not that the
   conversion failed. Don't chase them as conversion bugs.

## How to tell the conversion succeeded

- `npm install` and `npm run lint` both pass.
- The home page loads with **no `Failed to resolve module specifier`** errors
  (importmap good) and `config.json` is fetched.
- A PLP draft renders a product grid with facets; a PDP draft renders the
  product containers. Any remaining missing *data* points at the backend
  (Gotcha 3), not the code.
