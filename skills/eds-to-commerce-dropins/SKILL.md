---
name: eds-to-commerce-dropins
description: >-
  Add Adobe Commerce drop-ins (PDP, PLP, cart, mini-cart, checkout, account,
  auth, wishlist) to an existing Adobe Experience Manager Edge Delivery Services
  project, using hlxsites/aem-boilerplate-commerce as the source of components.
  Works for ANY existing EDS site — a pristine aem-boilerplate or a live project
  with custom blocks, a customized scripts.js/head.html, and its own design
  system. Use whenever someone wants to turn an EDS site into a storefront, "add
  commerce to my EDS project", wire up Catalog Service / Live Search drop-ins, or
  migrate to the commerce boilerplate — even if they don't say "drop-in" by name.
  There is no official upgrade path, so follow this procedure.
---

# Adding Commerce drop-ins to an existing EDS project

The vanilla [aem-boilerplate](https://github.com/adobe/aem-boilerplate) and the
[aem-boilerplate-commerce](https://github.com/hlxsites/aem-boilerplate-commerce)
are independent templates with no shared `git` history, so there is no "upgrade"
command. Converting means **layering** the commerce boilerplate's drop-in
machinery onto the target project. The target is rarely pristine — it usually has
custom blocks, a customized `scripts.js`, its own `head.html`, fonts, analytics,
and a design system. So the real work is **merging at a few specific seams**, not
copying files wholesale. This skill is that procedure, validated on a real
conversion.

**Mental model.** Commerce = vanilla EDS *plus* three layers you graft on:

1. **Drop-in packages** — `@dropins/*` npm deps whose built assets are
   **generated** into `scripts/__dropins__/` by a `postinstall` step.
2. **Module resolution** — an **importmap** in `head.html` that maps
   `@dropins/<pkg>/` to those generated files.
3. **A commerce runtime** — `scripts/commerce.js` (config + GraphQL fetch) plus
   per-drop-in initializers, hooked into the page lifecycle from `scripts.js`,
   and surfaced through blocks (`product-details`, `product-list-page`,
   `commerce-*`).

Your job is to install layer 1, declare layer 2, and **integrate** layer 3 into
the project's existing lifecycle and UI without breaking what's already there.

## The core decision: ADD vs MERGE vs REPLACE

For every file the commerce boilerplate provides, classify it against the target:

- **ADD** — file does not exist in the target. Copy it in as-is. (All
  `@dropins`-specific files: `commerce.js`, `initializers/`, `components/`,
  `build.mjs`, `postinstall.js`, `.npmrc`, and every `commerce-*` /
  `product-*` block.)
- **MERGE** — file exists and the target has customized it. Integrate the
  commerce additions by hand, preserving the target's code. (`scripts.js`,
  `head.html`, `package.json`, `styles/*.css`, `delayed.js`, lint configs,
  `.hlxignore`, and the shared `header`/`footer` blocks.)
- **REPLACE-if-pristine** — file exists but the target hasn't touched it (it's
  byte-for-byte the boilerplate version). Then a copy is safe. `scripts/aem.js`
  is the common case — but verify with `git log`/`diff` before replacing, and
  never assume.

When unsure, treat it as MERGE. The cost of an unnecessary merge is a few minutes;
the cost of an unnecessary replace is silently deleting the project's work.

## Step 0 — Take inventory (do this first)

Before touching anything, learn what the target customized. This determines how
much is ADD vs MERGE:

```bash
ls blocks/                                  # which custom blocks exist
git log --oneline -- scripts/scripts.js scripts/aem.js head.html styles/  # touched?
diff <(curl -s https://raw.githubusercontent.com/adobe/aem-boilerplate/main/scripts/aem.js) scripts/aem.js  # pristine aem.js?
```

Note: customized `scripts.js`? custom `header`/`footer`? a real design system in
`styles/`? Universal Editor models? Each "yes" is a MERGE, not a copy.

Always work on a feature branch.

## Step 1 — Clone the source of truth

```bash
git clone --depth 1 https://github.com/hlxsites/aem-boilerplate-commerce.git /tmp/abc-src
```

Reference `/tmp/abc-src` for every step; re-clone if it disappears. Pin to a known
release for reproducibility (validated against package **9.0.0**:
`storefront-pdp@3.2.0`, `storefront-cart@3.3.0`, `storefront-wishlist@3.3.0`,
`tools@1.9.0`).

## Step 2 — Dependencies + build machinery (MERGE package.json)

ADD `.npmrc`, `build.mjs`, `postinstall.js`. MERGE `package.json`:

- Add the source's `dependencies` (15: `@adobe/adobe-client-data-layer`, the two
  `@adobe/magento-storefront-event-*`, the eleven `@dropins/storefront-*`, and
  `@dropins/tools`) and `overrides`.
- Merge `scripts`: add `install:dropins`, `postinstall`, `postupdate`, and the
  `build:json*` targets. **Watch for collisions** — if the target already has a
  `postinstall` or `build`, chain them (`"postinstall": "existing && npm run
  install:dropins"`) rather than overwrite.
- Merge `devDependencies` (build-tools, husky, merge-json-cli, npm-run-all, etc.),
  keeping the target's existing tooling.
- **Keep the target's identity**: `name`, `version`, `description`, `repository`,
  `private`, `author`, `license`. Do not let upstream's `version: 9.0.0` overwrite
  the project's own.

```bash
mkdir -p scripts/acdl        # see Gotcha 1 — BEFORE npm install
npm install                  # runs postinstall → generates drop-in assets
```

Expect `✅ Drop-ins installed successfully!`.

## Step 3 — Commerce runtime (mostly ADD)

ADD these (they don't exist in a non-commerce project): `scripts/commerce.js`,
`scripts/ue.js`, `scripts/ue-utils.js`, `scripts/initializers/`,
`scripts/components/`. `commerce.js` reads store config and exposes
`CORE_FETCH_GRAPHQL` (core Commerce) and `CS_FETCH_GRAPHQL` (Catalog Service /
Live Search); the initializers register each drop-in on the shared event bus.

## Step 4 — Integrate the lifecycle (MERGE scripts.js)

This is the seam that makes drop-ins initialize. In a customized `scripts.js`,
**do not replace** — add these four things, preserving the project's own
autoblocking/decoration:

1. **Imports** from commerce.js:

   ```js
   import {
     loadCommerceEager, loadCommerceLazy, initializeCommerce,
     applyTemplates, decorateLinks, loadErrorPage,
     decorateSections, IS_UE, IS_DA,
   } from './commerce.js';
   ```

2. In **`decorateMain(main)`**, add `decorateLinks(main);` (commerce rewrites
   product/category links).

3. In **`loadEager(doc)`**, wrap the main decoration so commerce config loads
   before the first section, falling back to a 418 page on failure:

   ```js
   const main = doc.querySelector('main');
   if (main) {
     try {
       await initializeCommerce();
       decorateMain(main);          // the project's existing decorateMain
       applyTemplates(doc);
       await loadCommerceEager();
     } catch (e) {
       console.error('Error initializing commerce configuration:', e);
       loadErrorPage(418);
     }
     document.body.classList.add('appear');
     await loadSection(main.querySelector('.section'), waitForFirstImage);
   }
   ```

4. In **`loadLazy(doc)`**, add `loadCommerceLazy();` after the header/footer load.

If `scripts.js` is pristine, you may REPLACE it with the source's — but most live
projects have customized `buildAutoBlocks`/`decorateMain`, so merge.

## Step 5 — Module resolution (MERGE head.html)

The drop-ins are bare ES module imports (`@dropins/storefront-cart/...`); they
only resolve because of the importmap. **Merge into the existing `head.html`** (do
not replace — the project has its own meta/CSP/analytics/fonts):

- The `<script type="importmap">` mapping every `@dropins/<pkg>/` →
  `/scripts/__dropins__/<pkg>/`.
- `<script type="module" src="/scripts/commerce.js">`.
- The importmap shim `<script>` for browsers without importmap support.
- The `<link rel="modulepreload">` hints for the drop-in tools.

A `Failed to resolve module specifier "@dropins/..."` error at runtime means this
merge is missing or wrong — not that the packages failed to install.

## Step 6 — Styles (MERGE)

The commerce **blocks** carry their own CSS (added with the blocks in Step 7) and
the drop-in components ship their own design system, so you do **not** wholesale-
replace the project's `styles/styles.css`. Instead:

- Pull in any global CSS custom properties (design tokens) that commerce blocks
  reference but the project lacks, appending them rather than overwriting the
  project's tokens.
- Watch for **token name collisions** (e.g. `--color-*`, spacing, font vars): if
  both define the same variable, the commerce components may inherit the project's
  values — usually fine, but verify visually.
- Merge `lazy-styles.css`/`fonts.css` additively. Validate the storefront visually
  after, since this is where regressions hide.

## Step 7 — Blocks (ADD commerce blocks; MERGE header/footer)

ADD all `commerce-*`, `product-details`, `product-list-page`,
`product-recommendations`, and the support blocks (`accordion`, `carousel`,
`modal`, `enrichment`, `targeted-block`). They don't collide with custom blocks.

```bash
cp -R /tmp/abc-src/blocks/commerce-*           blocks/
cp -R /tmp/abc-src/blocks/product-*            blocks/
cp -R /tmp/abc-src/blocks/{accordion,carousel,modal,enrichment,targeted-block} blocks/
```

**`header`/`footer` are the biggest manual effort** because the project almost
certainly customized them. The commerce header adds a mini-cart, search, and
account/auth entry points. Two strategies:

- **Integrate (recommended for sites with significant custom nav):** keep the
  project's header markup and mount the commerce pieces into it — import the
  mini-cart and search drop-in containers and render them into the existing nav,
  cloning the wiring from the source `blocks/header/header.js`.
- **Adopt + reskin (fastest for simple headers):** take the source
  `header`/`footer` blocks wholesale and restyle them to match the brand.

ADD `icons/` (commerce SVGs) without removing the project's icons. If the project
uses Universal Editor, MERGE `models/` and run `npm run build:json` to regenerate
`component-{definition,models,filters}.json`.

## Step 8 — Lint config + .hlxignore (MERGE)

Add `scripts/__dropins__` to the project's `.eslintignore` (generated code isn't
linted) and apply any rule relaxations the commerce `.eslintrc.js` needs, rather
than replacing the project's lint config. Ensure `.hlxignore` keeps the project's
entries and adds `*.map` and `_*.json`. Run `npm run lint` — fix to exit 0.

## Step 9 — Backend config

Drop-ins read store config at runtime via
`@dropins/tools/lib/aem/configs.js`. Locally, serve it as a `config.json` at the
project root (`aem up` exposes it at `/config.json`); in production it comes from
the published `configs` content sheet. Shape:

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
endpoints are often the same unified GraphQL URL with no `x-api-key`. A classic
Adobe Commerce + separate Catalog Service tenant needs `x-api-key` and
`Magento-Environment-Id` in the `cs` headers.

## Step 10 — Validate locally

```bash
npx -y @adobe/aem-cli up --no-open --forward-browser-logs   # add --port N if 3000 is taken
```

If the backend has no authored commerce pages yet, validate with **draft pages**
served from a local folder (`--html-folder drafts`; the dev server serves them
under a `/drafts` prefix):

- **PDP** reads its SKU from a `/products/<urlKey>/<sku>` URL (or a `sku` meta).
  Create `drafts/products/<urlKey>/<SKU>.plain.html` with
  `<div><div class="product-details"></div></div>`.
- **PLP** uses Live Search; create `drafts/search.plain.html` with a
  `product-list-page` block.

Confirm: importmap resolves (no module-specifier errors), `config.json` is
fetched, the PLP grid renders with facets, and PDP containers populate. **Then
regression-check the project's own pages** — custom blocks, header/footer, and
styling should still work. That regression pass is the part the wholesale-replace
approach skips and breaks.

## Step 11 — Author a demo page and share a preview link

Local drafts prove the code, but reviewers (and the EDS PR convention) want a
working page on the real **branch preview** — `https://<branch>--<repo>--<owner>.aem.page/<path>`.
Content is **not** branch-specific: the same authored page renders against
whichever branch's code you preview, so author the page once in the content
source, then preview it on the feature branch's ref.

For a **document-authoring (DA / da.live) site** at
`https://da.live/#/<owner>/<repo>`:

1. **Author the page.** In `da.live`, create the page (e.g. `/shop`) and add the
   commerce block as a table — the first row is the block name, following rows are
   key/value config. A PLP page is one table:

   | Product List Page |
   | --- |
   | pageSize | 12 |

   (If you have the DA MCP / API connected, write the page HTML to
   `https://admin.da.live/source/<owner>/<repo>/<path>.html` instead of clicking.)

   **PDP works differently — there is ONE page, not one per product.** Do NOT
   author a page per SKU. Author a single template at **`/products/default`** (the
   value in `PRODUCT_TEMPLATE_PATHS` in `commerce.js`) holding a `Product Details`
   block with a `defaultSku` config row (used when editing in UE/DA). Then make
   real `/products/<urlKey>/<sku>` URLs render it via **folder mapping** in the
   site config (`/products/` → `/products/default`). Because folder mapping skips
   SEO, the boilerplate also imports a **bulk-metadata** sheet (`metadata.json`,
   generated by `tools/pdp-metadata`) that injects each product's
   `<meta name="sku">`, Open Graph, and JSON-LD server-side so crawlers and
   `getProductSku()` resolve the right product. On the rendered page,
   `getProductSku()` = `getMetadata('sku')` (from bulk metadata) `|| <sku from the
   URL>`. So the full authored PDP needs three things, none of which is a
   per-product page: the `products/default` template, folder mapping, and the
   bulk-metadata sheet.

2. **Preview on the branch ref** so it picks up your branch's code:
   `POST https://admin.hlx.page/preview/<owner>/<repo>/<branch>/<path>`
   (or click **Preview** in the DA sidebar, then open the branch preview URL).

3. **Share the link** in the PR description:
   `https://<branch>--<repo>--<owner>.aem.page/<path>`. Prefer the **PLP** page as
   the primary demo — it renders fully. If you include a PDP and the product data
   is blank, link it with the Gotcha-3 backend note so reviewers don't read it as a
   conversion bug.

For Google/SharePoint-backed sites the idea is identical — author the page in the
doc source, preview on the branch ref, share the branch preview URL.

## File manifest

| File(s) | Action |
|---|---|
| `.npmrc`, `build.mjs`, `postinstall.js`, `scripts/commerce.js`, `scripts/ue*.js`, `scripts/initializers/`, `scripts/components/`, all `commerce-*`/`product-*`/support blocks, commerce `icons/` | **ADD** |
| `package.json`, `scripts/scripts.js`, `head.html`, `styles/*.css`, `scripts/delayed.js`, `.eslintrc.js`, `.eslintignore`, `.stylelintrc.json`, `.hlxignore`, `blocks/header`, `blocks/footer`, `models/` | **MERGE** |
| `scripts/aem.js` (and `scripts.js`/`head.html` only if untouched) | **REPLACE-if-pristine** (verify with `git diff` first) |
| `scripts/__dropins__/**`, `scripts/commerce-events-*.js`, `scripts/acdl/**`, `component-*.json` | **GENERATE** (never hand-copy) |
| the project's custom blocks, fonts, content, identity fields | **KEEP** |

## Generated vs. copied (the most common mistake)

The GENERATE row is produced by `npm install` (runs `postinstall.js`, copying
built bundles out of `node_modules/@dropins/*`) and `npm run build:json` (merges
`models/_*.json`). Never copy them from the source clone or edit by hand — they'd
drift from your installed versions. They *are* committed to git and *are* served
by the CDN (they're the runtime bundles); they're only excluded from linting.
Update drop-ins later with `npm update @dropins/...` then `npm run postupdate`.

## Gotchas (verified)

1. **Pre-create `scripts/acdl/` before the first `npm install`.** `postinstall.js`
   uses `fs.copyFileSync` for the Adobe Client Data Layer, which doesn't create
   parent dirs — a first run crashes with `ENOENT`. `mkdir -p scripts/acdl` first,
   or run `npm install` twice.

2. **Don't add `__dropins__` to `.hlxignore`.** Generated drop-ins are meant to be
   served; they're excluded from linting via `.eslintignore` only.

3. **Empty PDP / `$0.00` prices are almost always a backend scoping problem, not a
   code bug.** Drop-ins attach a `Magento-Customer-Group` header (anonymous =
   `sha1("0")` = `b6589fc6ab0dc82cf12099d1c2d40ab994e8410c`). If the tenant's
   Catalog Service / price book isn't indexed for that group, `products(skus:)`
   returns `{ "data": { "products": [] } }` with HTTP 200 and the PDP renders an
   empty shell. Diagnose by replaying the GraphQL request as `curl` **with and
   without** the `Magento-Customer-Group` header — if removing it returns the
   product, fix the backend (index the catalog/price book for that group). Live
   Search (`productSearch`, the PLP) is unaffected.

4. **Port 3000 may be busy.** `aem up` won't fall back; pass `--port N`.

5. **Console 404s for `placeholders/*.json`, `nav.plain.html`,
   `footer.plain.html`, and fonts are expected on a fresh backend** — content/fonts
   aren't authored yet, not a conversion failure.

6. **`scripts.js`/`head.html` merges are where conversions silently break.** If
   drop-ins never initialize, re-check the four `scripts.js` seams (Step 4); if
   imports don't resolve, re-check the importmap merge (Step 5).

## Success criteria

- `npm install` and `npm run lint` pass.
- Home page loads with no `Failed to resolve module specifier` errors and fetches
  `config.json`.
- A PLP draft renders a product grid with facets; a PDP draft renders its
  containers. Remaining missing *data* points at the backend (Gotcha 3).
- **The project's pre-existing pages, blocks, and styling still work** — the
  conversion added commerce without regressing the site.
