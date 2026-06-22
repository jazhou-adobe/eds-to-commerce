# EDS → Commerce Drop-in Conversion — Design

**Date:** 2026-06-22
**Status:** Approved for planning
**Author:** James Zhou (with Claude)

## Goal

Convert this vanilla `aem-boilerplate` project into a Commerce-drop-in-enabled
storefront, mirroring `hlxsites/aem-boilerplate-commerce`. Then capture the exact
conversion procedure as a shareable skill (`SKILL.md`) so others can repeat it on
their own EDS projects — there is no official upgrade path today, so the skill is
the durable artifact.

**Primary order:** convert this repo first (live, validated), then back-fill the skill
from what was actually done.

## Scope decisions (confirmed)

| Decision | Choice |
|---|---|
| Deliverable order | Convert repo first, then write the skill |
| Storefront scope | **Full storefront** — PLP, PDP, recommendations, cart, mini-cart, checkout, order, account, auth, wishlist, payment-services |
| Ingest method | **Copy files manually** from a clone of `aem-boilerplate-commerce` |
| Backend | **Sandbox demo backend** using the user-provided config (below) |
| Workspace | Feature branch `convert-to-commerce-dropins` on this repo |
| Versions | **Latest compatible** — keep the boilerplate's `^` ranges; `npm install` resolves latest compatible |

## Source of truth

`hlxsites/aem-boilerplate-commerce` @ `main` (cloned to `/tmp/abc-commerce` for this
work; currently package version 9.0.0). Drop-in packages are at `@dropins/storefront-*`
v3–4 ranges. Because we chose latest-compatible, we copy the boilerplate's `package.json`
dependency ranges verbatim and let `npm install` resolve.

## Starting state

This repo is a **clean, unmodified vanilla `aem-boilerplate`** on `main`:
- blocks: `cards`, `columns`, `footer`, `fragment`, `header`, `hero`, `widget`
- scripts: `aem.js`, `scripts.js`, `delayed.js`
- styles: `styles.css`, `lazy-styles.css`, `fonts.css`
- minimal `head.html`, standard lint configs, no build step

Because there is no custom application code, infrastructure files can be **replaced
wholesale** rather than carefully merged. The only bespoke block is `widget`, which is
inert and will be kept.

## Architecture of the target (what Commerce adds)

1. **Build/dependency layer** — `@dropins/*` + `@adobe/*` npm packages. A `postinstall`
   step (`build.mjs` + `postinstall.js`) copies built drop-in assets from
   `node_modules/@dropins/*` into `scripts/__dropins__/` and the event SDKs into
   `scripts/commerce-events-*.js` + `scripts/acdl/`. **These outputs are generated, not
   hand-authored.** `build.mjs` runs `@dropins/build-tools` to trim GraphQL operations
   unsupported by ACCS.

2. **Script wiring layer** — `scripts/commerce.js` (config reader via
   `@dropins/tools/lib/aem/configs.js`, `CORE_FETCH_GRAPHQL`/`CS_FETCH_GRAPHQL` helpers,
   commerce decorators, `IS_UE`/`IS_DA` flags). `scripts/scripts.js` is the Commerce
   variant that calls `loadCommerceEager/Lazy` and `initializeCommerce`.
   `scripts/initializers/*` (one per drop-in) register each drop-in on the event bus.

3. **Module resolution** — `head.html` declares an importmap mapping `@dropins/<pkg>/`
   → `/scripts/__dropins__/<pkg>/`, plus modulepreloads and the `commerce.js` module
   script, plus a shim for browsers without importmap support.

4. **Blocks layer** — 44 blocks: storefront drop-in containers (`product-list-page`,
   `product-details`, `product-recommendations`, `commerce-*`) + supporting UI blocks
   (`accordion`, `carousel`, `modal`, `enrichment`, `targeted-block`) + Commerce
   `header`/`footer` (mini-cart + search wired).

5. **Authoring/UE layer** — `models/` source fragments, merged by `build:json` into
   `component-{definition,models,filters}.json` for Universal Editor.

6. **Config layer** — drop-ins read store config (endpoints, headers, analytics) at
   runtime via the `configs` mechanism. Locally this must resolve to the sandbox config.

## File manifest (copy / replace / generate / keep)

**Replace (wholesale):**
- `scripts/scripts.js`, `scripts/aem.js`
- `head.html`
- `styles/styles.css`, `styles/lazy-styles.css`, `styles/fonts.css`
- `.hlxignore`, `.eslintrc.js`, `.eslintignore`, `.stylelintrc.json`
- `package.json` (deps/devDeps/scripts/overrides; keep project `name`)

**Copy (new):**
- `scripts/commerce.js`, `scripts/ue.js`, `scripts/ue-utils.js`
- `scripts/initializers/` (12 files), `scripts/components/commerce-mini-pdp/`
- `.npmrc`, `build.mjs`, `postinstall.js`
- all Commerce `blocks/*` (44), `icons/`, `models/`, `418.html`

**Generate (via `npm install` / `npm run build:json` — never hand-copy):**
- `scripts/__dropins__/**`, `scripts/commerce-events-collector.js`,
  `scripts/commerce-events-sdk.js`, `scripts/acdl/**`
- `component-definition.json`, `component-models.json`, `component-filters.json`

**Keep (this repo's):**
- `blocks/widget` (inert, no conflict), `blocks/cards|columns|hero` (compatible),
  `scripts/delayed.js` (reconcile with Commerce version if it differs),
  `favicon.ico`, `fonts/`, `AGENTS.md`/`CLAUDE.md`, LICENSE/README

**Open item to verify empirically:** Commerce `header`/`footer`/`fragment` replace this
repo's versions (mini-cart/search integration). Confirmed during execution by diffing.

## Backend config

Use the user-provided sandbox config (NA1 sandbox tenant):

```json
{
  "public": {
    "default": {
      "commerce-core-endpoint": "https://na1-sandbox.api.commerce.adobe.com/PtyNpE675vtkJo73YwDeA7/graphql",
      "commerce-endpoint": "https://na1-sandbox.api.commerce.adobe.com/PtyNpE675vtkJo73YwDeA7/graphql",
      "headers": {
        "all": { "Store": "default" },
        "cs": {
          "Magento-Store-Code": "main_website_store",
          "Magento-Store-View-Code": "default",
          "Magento-Website-Code": "base"
        }
      },
      "commerce-assets-enabled": true,
      "commerce-assets-quality": 80,
      "analytics": {
        "base-currency-code": "USD",
        "environment": "Production",
        "store-code": "main_website_store",
        "store-view-code": "default",
        "website-code": "base"
      }
    }
  }
}
```

**Wiring (to verify against the running dev server):** the drop-ins resolve config via
`@dropins/tools/lib/aem/configs.js`, which fetches a published `configs` sheet
(`/configs.json` style) relative to the code base path. For local validation we place
this JSON where the dev server / `getConfigValue` reads it (candidate locations:
`/config.json` at repo root served by `aem up`, or a local `configs` content file). The
exact path is the single empirically-confirmed step; the skill will document the
confirmed answer rather than a guess.

## Validation strategy

1. `npm install` succeeds; `scripts/__dropins__/` populated.
2. `npm run build:json` produces the three component JSON files.
3. `npm run lint` passes (Commerce eslint config ignores `__dropins__`).
4. `aem up` serves at `localhost:3000`; load via Playwright:
   - Home page renders, no console errors, importmap resolves.
   - PDP renders a product from the sandbox.
   - PLP renders a product grid.
   - Add-to-cart → mini-cart updates.
   - Cart page + checkout step load against the sandbox.
5. Capture every error/fix as a gotcha entry for the skill.

Done = full storefront renders against the sandbox with a clean console and passing lint.

## The skill (final deliverable)

After a validated conversion, author skill `eds-to-commerce-dropins` via skill-creator:
- **What it does:** converts a vanilla EDS `aem-boilerplate` project to Commerce drop-ins.
- **Contents:** ordered procedure; the file manifest above (copy/replace/generate/keep);
  the generated-vs-copied distinction (the most common mistake); backend-config wiring
  with the confirmed local path; version/pinning guidance and how to bump drop-ins later
  (`postupdate`); and the real gotchas hit during this conversion.
- **Form:** shareable `SKILL.md` with `name` + `description` frontmatter for trigger
  matching; reference the `aem-boilerplate-commerce` repo as upstream source of truth.

## Out of scope

- Production backend / real catalog (sandbox only).
- Custom storefront theming beyond what the boilerplate ships.
- Universal Editor authoring setup beyond generating the component JSON.
- Cypress E2E suite from the Commerce repo (we validate with Playwright manually).

## Risks

- **Config resolution path** for local dev is the main unknown — mitigated by empirical
  verification against the running server before writing the skill.
- **Latest-compatible drift** — a drop-in newer than v9.0.0's tested set could introduce
  a break; mitigated by the boilerplate's `^` ranges being the same set it ships with,
  and full local validation catching regressions.
- **header/footer/delayed.js divergence** — resolved by diffing during execution.
