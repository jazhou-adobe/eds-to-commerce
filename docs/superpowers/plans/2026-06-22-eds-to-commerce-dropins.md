# EDS → Commerce Drop-in Conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert this vanilla `aem-boilerplate` repo into a Commerce-drop-in storefront (full storefront) wired to the sandbox backend, validated locally, then capture the procedure as a shareable skill.

**Architecture:** Copy infrastructure from a fresh clone of `hlxsites/aem-boilerplate-commerce` into this repo, regenerate drop-in assets via its `postinstall` build step, serve the sandbox config as a root `config.json`, validate the storefront with lint + `aem up` + Playwright, then author the skill from the validated result.

**Tech Stack:** Edge Delivery Services, vanilla ES6 modules, `@dropins/storefront-*` packages, `@dropins/build-tools`, importmap module resolution, `@adobe/aem-cli` (`aem up`), Playwright for browser validation.

## Global Constraints

- Source of truth: `hlxsites/aem-boilerplate-commerce` @ `main` (clone fresh; do not rely on `/tmp/abc-commerce` persisting).
- Versions: **latest-compatible** — keep the boilerplate's `^` dependency ranges verbatim; let `npm install` resolve.
- Generated, NEVER hand-copied: `scripts/__dropins__/**`, `scripts/commerce-events-collector.js`, `scripts/commerce-events-sdk.js`, `scripts/acdl/**`, `component-definition.json`, `component-models.json`, `component-filters.json`.
- Work on branch `convert-to-commerce-dropins` (already created).
- Keep this repo's project `name` in `package.json`, `AGENTS.md`, `CLAUDE.md`, `LICENSE`, `README.md`, `favicon.ico`.
- Keep the inert custom block `blocks/widget`.
- Sandbox backend config (NA1) is the user-provided JSON — see Task 9, use verbatim.
- Convert + validate the repo FIRST; author the skill (Task 12) only after validation passes.

---

### Task 1: Fresh source clone

**Files:**
- Create: `/tmp/abc-commerce-src` (working clone, outside repo)

- [ ] **Step 1: Clone the Commerce boilerplate**

```bash
rm -rf /tmp/abc-commerce-src
git clone --depth 1 https://github.com/hlxsites/aem-boilerplate-commerce.git /tmp/abc-commerce-src
```

- [ ] **Step 2: Verify the clone has expected structure**

```bash
ls /tmp/abc-commerce-src/scripts/initializers && ls /tmp/abc-commerce-src/blocks | wc -l && cat /tmp/abc-commerce-src/package.json | grep '"version"'
```
Expected: lists 12 initializer files; block count ~44; a package version line prints.

---

### Task 2: Build/dependency machinery + generate drop-in assets

**Files:**
- Modify: `package.json` (deps/devDeps/scripts/overrides; keep `name`)
- Create: `.npmrc`, `build.mjs`, `postinstall.js`
- Generate: `scripts/__dropins__/**`, `scripts/commerce-events-*.js`, `scripts/acdl/**`

**Interfaces:**
- Produces: populated `scripts/__dropins__/<pkg>/` consumed by the importmap (Task 4) and initializers (Task 3).

- [ ] **Step 1: Copy build files**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp /tmp/abc-commerce-src/.npmrc .npmrc
cp /tmp/abc-commerce-src/build.mjs build.mjs
cp /tmp/abc-commerce-src/postinstall.js postinstall.js
```

- [ ] **Step 2: Merge dependency/script blocks into package.json**

Replace this repo's `scripts`, `devDependencies` with the Commerce ones and add `dependencies` + `overrides` from `/tmp/abc-commerce-src/package.json`. Keep this repo's `name`, `private`, `repository`, `author`, `license`. The resulting `package.json` must contain:
- `scripts`: `lint:js`, `lint:css`, `lint`, `lint:fix`, `start`, `install:dropins`, `postinstall`, `postupdate`, `build:json`, `build:json:models`, `build:json:definitions`, `build:json:filters`, `prepare`.
- `dependencies`: `@adobe/adobe-client-data-layer`, `@adobe/magento-storefront-event-collector`, `@adobe/magento-storefront-events-sdk`, `@dropins/storefront-account`, `-auth`, `-cart`, `-checkout`, `-order`, `-payment-services`, `-pdp`, `-personalization`, `-product-discovery`, `-recommendations`, `-wishlist`, `@dropins/tools`.
- `devDependencies`: `@adobe/aem-cli`, `@babel/eslint-parser`, `@dropins/build-tools`, `eslint`, `eslint-config-airbnb-base`, `eslint-plugin-import`, `husky`, `merge-json-cli`, `npm-run-all`, `stylelint`, `stylelint-config-standard`.
- `overrides`: the `@adobe/alloy` block.

Use the exact version ranges from `/tmp/abc-commerce-src/package.json`.

- [ ] **Step 3: Install (runs postinstall → generates drop-in assets)**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
npm install 2>&1 | tail -20
```
Expected: ends with `✅ Drop-ins installed successfully!` (no artifactory error).

- [ ] **Step 4: Verify generated assets exist**

```bash
ls scripts/__dropins__ && ls scripts/acdl && ls scripts/commerce-events-*.js
```
Expected: `__dropins__` lists `storefront-*` + `tools`; `acdl` has the min.js; both event JS files present.

- [ ] **Step 5: Commit (assets are gitignored-by-.hlxignore but committed to git)**

```bash
git add package.json package-lock.json .npmrc build.mjs postinstall.js scripts/__dropins__ scripts/acdl scripts/commerce-events-collector.js scripts/commerce-events-sdk.js
git commit -m "Add commerce dependencies and generated drop-in assets"
```

---

### Task 3: Script wiring

**Files:**
- Replace: `scripts/scripts.js`, `scripts/aem.js`, `scripts/delayed.js`
- Create: `scripts/commerce.js`, `scripts/ue.js`, `scripts/ue-utils.js`, `scripts/initializers/` (12 files), `scripts/components/commerce-mini-pdp/`

**Interfaces:**
- Consumes: `scripts/__dropins__/**` from Task 2.
- Produces: `loadCommerceEager`, `loadCommerceLazy`, `initializeCommerce`, `IS_UE`, `IS_DA` exported from `scripts/commerce.js`, imported by `scripts/scripts.js`.

- [ ] **Step 1: Copy script wiring files**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp /tmp/abc-commerce-src/scripts/scripts.js scripts/scripts.js
cp /tmp/abc-commerce-src/scripts/aem.js scripts/aem.js
cp /tmp/abc-commerce-src/scripts/delayed.js scripts/delayed.js
cp /tmp/abc-commerce-src/scripts/commerce.js scripts/commerce.js
cp /tmp/abc-commerce-src/scripts/ue.js scripts/ue.js
cp /tmp/abc-commerce-src/scripts/ue-utils.js scripts/ue-utils.js
cp -R /tmp/abc-commerce-src/scripts/initializers scripts/initializers
cp -R /tmp/abc-commerce-src/scripts/components scripts/components
```

- [ ] **Step 2: Verify imports resolve to copied files**

```bash
grep -c "from './commerce.js'" scripts/scripts.js && ls scripts/initializers | wc -l && ls scripts/components
```
Expected: `1`; `12`; `commerce-mini-pdp`.

- [ ] **Step 3: Commit**

```bash
git add scripts/scripts.js scripts/aem.js scripts/delayed.js scripts/commerce.js scripts/ue.js scripts/ue-utils.js scripts/initializers scripts/components
git commit -m "Add commerce script wiring and initializers"
```

---

### Task 4: head.html (importmap + module wiring)

**Files:**
- Replace: `head.html`

**Interfaces:**
- Consumes: `scripts/__dropins__/**` (Task 2), `scripts/commerce.js` + `scripts/initializers/*` (Task 3).

- [ ] **Step 1: Copy head.html**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp /tmp/abc-commerce-src/head.html head.html
```

- [ ] **Step 2: Verify importmap + commerce.js script present**

```bash
grep -c "@dropins/storefront-cart/" head.html && grep -c "scripts/commerce.js" head.html
```
Expected: `1` and `1`.

- [ ] **Step 3: Commit**

```bash
git add head.html
git commit -m "Add commerce importmap and module wiring to head.html"
```

---

### Task 5: Blocks (full storefront)

**Files:**
- Create/Replace: all of `/tmp/abc-commerce-src/blocks/*` copied into `blocks/` (overwrites `cards`, `columns`, `hero`, `header`, `footer`, `fragment`; adds 38+ commerce/support blocks). Keeps `blocks/widget`.

- [ ] **Step 1: Copy all commerce blocks**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp -R /tmp/abc-commerce-src/blocks/. blocks/
```

- [ ] **Step 2: Verify storefront + preserved blocks**

```bash
ls blocks | grep -E "product-details|product-list-page|commerce-cart|commerce-checkout|commerce-mini-cart" && ls blocks/widget
```
Expected: all five storefront blocks listed; `widget` still present.

- [ ] **Step 3: Commit**

```bash
git add blocks
git commit -m "Add commerce storefront blocks"
```

---

### Task 6: Styles + icons

**Files:**
- Replace: `styles/styles.css`, `styles/lazy-styles.css`, `styles/fonts.css`
- Create/Replace: `icons/`

- [ ] **Step 1: Copy styles and icons**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp /tmp/abc-commerce-src/styles/styles.css styles/styles.css
cp /tmp/abc-commerce-src/styles/lazy-styles.css styles/lazy-styles.css
cp /tmp/abc-commerce-src/styles/fonts.css styles/fonts.css
cp -R /tmp/abc-commerce-src/icons/. icons/
```

- [ ] **Step 2: Verify commerce design tokens landed**

```bash
grep -c -- "--color-" styles/styles.css
```
Expected: a non-zero count (commerce design tokens present).

- [ ] **Step 3: Commit**

```bash
git add styles icons
git commit -m "Add commerce styles and icons"
```

---

### Task 7: Models + generated component JSON (Universal Editor)

**Files:**
- Create: `models/`
- Generate: `component-definition.json`, `component-models.json`, `component-filters.json`

- [ ] **Step 1: Copy models source**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp -R /tmp/abc-commerce-src/models models
```

- [ ] **Step 2: Generate component JSON**

```bash
npm run build:json 2>&1 | tail -10
```
Expected: completes without error.

- [ ] **Step 3: Verify generated files**

```bash
ls component-definition.json component-models.json component-filters.json
```
Expected: all three files present.

- [ ] **Step 4: Commit**

```bash
git add models component-definition.json component-models.json component-filters.json
git commit -m "Add commerce component models and generated definitions"
```

---

### Task 8: Lint configs, .hlxignore, 418.html

**Files:**
- Replace: `.hlxignore`, `.eslintrc.js`, `.eslintignore`, `.stylelintrc.json`
- Create: `418.html`

- [ ] **Step 1: Copy config files**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
cp /tmp/abc-commerce-src/.hlxignore .hlxignore
cp /tmp/abc-commerce-src/.eslintrc.js .eslintrc.js
cp /tmp/abc-commerce-src/.eslintignore .eslintignore
cp /tmp/abc-commerce-src/.stylelintrc.json .stylelintrc.json
cp /tmp/abc-commerce-src/418.html 418.html
```

- [ ] **Step 2: Verify .hlxignore ignores generated/source-only files**

```bash
grep -E "\*\.map|__dropins__|postinstall|build.mjs|_\*\.json" .hlxignore
```
Expected: matches present (at minimum `*.map` and `_*.json`).

- [ ] **Step 3: Commit**

```bash
git add .hlxignore .eslintrc.js .eslintignore .stylelintrc.json 418.html
git commit -m "Add commerce lint configs, hlxignore, and 418 page"
```

---

### Task 9: Backend config (sandbox)

**Files:**
- Create: `config.json` (repo root, served at `/config.json`)

- [ ] **Step 1: Write config.json with the sandbox config**

Create `config.json` with exactly:

```json
{
  "public": {
    "default": {
      "commerce-core-endpoint": "https://na1-sandbox.api.commerce.adobe.com/PtyNpE675vtkJo73YwDeA7/graphql",
      "commerce-endpoint": "https://na1-sandbox.api.commerce.adobe.com/PtyNpE675vtkJo73YwDeA7/graphql",
      "headers": {
        "all": {
          "Store": "default"
        },
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

- [ ] **Step 2: Verify it is valid JSON**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
node -e "JSON.parse(require('fs').readFileSync('config.json','utf8')); console.log('valid')"
```
Expected: `valid`.

- [ ] **Step 3: Commit**

```bash
git add config.json
git commit -m "Add sandbox commerce backend config"
```

---

### Task 10: Lint validation

- [ ] **Step 1: Run lint**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
npm run lint 2>&1 | tail -30
```
Expected: exits 0. If failures are in copied commerce code, run `npm run lint:fix`; if failures are in generated `__dropins__`, confirm `.eslintignore`/`.hlxignore` excludes them and re-run. Capture any fix as a skill gotcha.

- [ ] **Step 2: Commit any lint fixes**

```bash
git commit -am "Fix lint after commerce conversion" || echo "no lint fixes needed"
```

---

### Task 11: Dev server + browser validation

**Interfaces:**
- Consumes: everything above. This is the end-to-end gate.

- [ ] **Step 1: Start the dev server (background)**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
npx -y @adobe/aem-cli up --no-open --forward-browser-logs
```
Run in background. Expected: serves at `http://localhost:3000`.

- [ ] **Step 2: Confirm config.json is served**

```bash
curl -s http://localhost:3000/config.json | node -e "let d='';process.stdin.on('data',c=>d+=c).on('end',()=>{JSON.parse(d);console.log('served+valid')})"
```
Expected: `served+valid`. If 404, the local config path is wrong — adjust filename/location until `getConfigValue` resolves it, and record the working answer for the skill (and update the spec).

- [ ] **Step 3: Load home page in Playwright, check console**

Navigate Playwright to `http://localhost:3000/`. Capture console messages. Expected: page renders, importmap resolves (no "Failed to resolve module specifier @dropins/..."), no uncaught errors.

- [ ] **Step 4: Validate PDP**

Navigate to a product detail page path served by the sandbox (discover a valid product/SKU via the storefront or a PLP link). Expected: product title, price, gallery, add-to-cart button render. If Catalog Service calls 401, flag the missing `x-api-key`/`Magento-Environment-Id` headers to the user (see spec risk).

- [ ] **Step 5: Validate PLP**

Navigate to a product-list / category page. Expected: a product grid renders with items from the sandbox.

- [ ] **Step 6: Validate cart + mini-cart + checkout**

Add a product to cart from PDP. Expected: mini-cart updates with the item. Navigate to the cart page and the checkout block. Expected: both render against the sandbox without console errors.

- [ ] **Step 7: Record results**

Note every error encountered and its fix in a scratch list for Task 12. Done = storefront renders end-to-end with a clean console.

---

### Task 12: Author the shareable skill

**Files:**
- Create: skill directory + `SKILL.md` (via skill-creator)

**Interfaces:**
- Consumes: the validated conversion + the gotcha list from Task 11.

- [ ] **Step 1: Invoke skill-creator**

Use the `skill-creator` skill to scaffold a new skill named `eds-to-commerce-dropins`.

- [ ] **Step 2: Author SKILL.md content**

`SKILL.md` must include, in order:
1. Frontmatter `name: eds-to-commerce-dropins` + a trigger `description` (e.g. "convert a vanilla AEM EDS aem-boilerplate project to Adobe Commerce drop-ins").
2. Upstream source-of-truth pointer: `hlxsites/aem-boilerplate-commerce` (clone fresh).
3. The ordered procedure = Tasks 1–11 of this plan, condensed to actionable steps.
4. The file manifest table (copy / replace / generate / keep) from the spec.
5. **Generated-vs-copied** call-out (the most common mistake): `__dropins__`, `commerce-events-*.js`, `acdl/`, and the three `component-*.json` are produced by `npm install` + `npm run build:json`, never hand-copied.
6. Backend config wiring: root `config.json` shape + the confirmed local-serving behavior from Task 11 Step 2.
7. Version/upgrade guidance: keep `^` ranges; bump drop-ins later via `npm update` + `npm run postupdate`.
8. The real gotchas captured in Task 11.

- [ ] **Step 3: Verify the skill is well-formed**

Confirm `SKILL.md` has valid frontmatter and no placeholders/TODOs.

- [ ] **Step 4: Commit**

```bash
cd /Users/jazhou/Git/Workspace/SiteMigration/eds-to-commerce
git add <skill-path>
git commit -m "Add eds-to-commerce-dropins conversion skill"
```

---

## Self-Review

**Spec coverage:** Build machinery → T2. Script wiring → T3. head.html → T4. Blocks → T5. Styles/icons → T6. Models/UE JSON → T7. Lint/hlxignore/418 → T8. Backend config → T9. Validation strategy → T10–T11. Skill deliverable → T12. All spec sections covered.

**Placeholder scan:** Concrete commands and the full `config.json` body are inline. The one intentionally-discovered value is a valid sandbox product path/SKU in T11 (cannot be known until the storefront/sandbox is queried) — discovery is part of that step, not a placeholder.

**Type consistency:** `loadCommerceEager`/`loadCommerceLazy`/`initializeCommerce`/`IS_UE`/`IS_DA` are produced in T3 and consumed by the copied `scripts.js` (same file set, so signatures match upstream by construction). Generated-asset names are identical across Global Constraints, T2, and T12.
