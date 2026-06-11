# UE Storybook Audit

Date: 2026-06-09
Workspace: `/Users/rayelo/G2/my-g2`
UE checkout inspected: `/Users/rayelo/G2/my-g2/ue`

## Executive Summary

UE has one active root Storybook setup: `.storybook/main.js`, wired to `webpack/doc/**/*.stories.mdx` and `webpack/spec/**/*.stories.js`, with package scripts on port `3091`.

It does not currently produce a working local Storybook preview in this checkout. `yarn storybook:dev` starts Storybook 6.5.16 and serves HTTP 200 on `http://localhost:3091/`, but the preview Webpack build is broken. `yarn build-storybook` fails with the same preview build class of errors.

The active story set is mostly a legacy/review-form documentation and state-catalog tool: Phoenix review form stories, older legacy review form stories, a few composite examples, and utility atomics docs. Maintenance signals are weak: Storybook CI was removed, ViewComponent Storybook was dropped, docs reference stale scripts, and recent commits touch these files incidentally through review-form or dependency changes rather than active Storybook ownership.

Recommendation: do not treat the current Storybook as a healthy supported surface. Either formally repair and own it for review-form state development, or deprecate it and migrate any still-useful examples into the current component/documentation path. The practical near-term recommendation is "repair only if the review-form team still wants this workflow; otherwise deprecate."

## Facts

### Instance Count

Active root setup:

- Config files: `.storybook/main.js`, `.storybook/preview.js`, `.storybook/preview-head.html`, `.storybook/webpack.config.js`, `.storybook/README.md`
- Package scripts:
  - `storybook:start`: `start-storybook -c .storybook -p 3091`
  - `storybook:dev`: `npm-run-all -p storybook:start`
  - `build-storybook`: `node script/update-storybook-css.js && build-storybook`
- Story globs in `.storybook/main.js`:
  - `../webpack/doc/Intro.stories.mdx`
  - `../webpack/doc/**/*.stories.mdx`
  - `../webpack/spec/**/*.stories.js`

No other runnable Storybook package/config appears active in the checkout:

- `spec/components/stories/application.stories.json` exists but is outside the active root globs.
- `view_component_storybook/.storybook/*` and `view_component_storybook/package.json` were deleted in `7bddd9773a chore: Drop ViewComponent storybook (#33179)`.
- `lib/gem_ext/view_component_storybook/*` still exists and is required by `lib/gem_ext.rb`, but it is not a separate runnable Storybook app/config.
- `engines/elevate/AGENTS.md` mentions Lookbook as "Storybook preview" at `/lookbook`; that is a separate Lookbook/ViewComponent documentation surface, not this React Storybook.

### Version

From `package.json`:

- `@storybook/react`: `^6.5.10`
- Storybook addons: `@storybook/addon-a11y`, `addon-docs`, `addon-essentials`, `addon-interactions`, `addon-measure`: `^6.5.10`
- `@storybook/addon-postcss`: `^2.0.0`
- `storybook-addon-pseudo-states`: `^1.15.1`
- `storybook-addon-recoil-flow`: `^0.0.5`

From `yarn.lock` and command output:

- `@storybook/react@npm:^6.5.10` resolves to `6.5.16`
- The dev/build command banner reports `@storybook/react v6.5.16`
- Core listed Storybook addons also resolve to `6.5.16` in `yarn.lock`, except `@storybook/addon-postcss@2.0.0`

### How To Run Locally

Use the UE login shell/mise environment, not a bare non-login shell:

```bash
cd /Users/rayelo/G2/my-g2/ue
zsh -lic 'yarn storybook:dev'
```

Equivalent direct script:

```bash
cd /Users/rayelo/G2/my-g2/ue
zsh -lic 'yarn storybook:start'
```

Expected port:

```text
http://localhost:3091/
```

Style/assets prerequisite:

- `.storybook/main.js` serves `../public/assets` at `/assets`.
- `.storybook/preview-head.html` references `/assets/nessy_app.css`.
- `.storybook/README.md` says local styles must already exist under `public/assets`.
- `script/update-storybook-css.js` expects `public/assets/nessy_app.css`, copies it to `.storybook/assets/nessy_app.css`, and rewrites `.storybook/preview-head.html` from `/assets/` to `./assets/` for static builds.

In this checkout, `public/assets/nessy_app.css` existed before verification. `.storybook/assets/nessy_app.css` did not exist until `build-storybook` generated it.

## Verification Results

Environment check:

```bash
zsh -lic 'ruby -v && node -v && yarn -v'
```

Result:

- Ruby: `3.3.10`
- Node: `v24.14.1`
- Yarn: `4.13.0`
- A non-login shell had `node v26.3.0` and no `yarn`, so Storybook commands should be run through the UE login shell setup.

### `yarn storybook:dev`

Command:

```bash
zsh -lic 'yarn storybook:dev'
```

Result: failed.

Observed behavior:

- Storybook starts and reports `@storybook/react v6.5.16`.
- It serves static files from `./public/assets` at `/assets`.
- `curl http://localhost:3091/` returned `200`.
- `curl http://localhost:3091/iframe.html` returned `200`.
- The preview build is broken, and Storybook prompts for crash-report opt-in before the script exits non-zero.

Representative errors:

- `ModuleDependencyError: Can't import the named export 'useState' from non EcmaScript module (only default export is available)`
- `ModuleParseError: Module parse failed: Unexpected token`, including modern syntax such as `??`
- Parse failures were observed in current dependency code including `node_modules/web-vitals/dist/web-vitals.js`, `node_modules/@testing-library/react/dist/@testing-library/react.esm.js`, and Amplitude/session replay dependency code.

Interpretation: the manager/server can answer on port `3091`, but the Storybook preview bundle is not usable.

### `yarn storybook:start`

`storybook:dev` is just `npm-run-all -p storybook:start`, so this exercised `storybook:start` as its only child command. The child command exited with:

```text
ERROR: "storybook:start" exited with 1.
```

### `yarn build-storybook`

Command:

```bash
zsh -lic 'yarn build-storybook'
```

Result: failed.

Observed behavior:

- `script/update-storybook-css.js` ran first.
- It generated `.storybook/assets/nessy_app.css`.
- It rewrote `.storybook/preview-head.html` from `/assets/nessy_app.css` to `./assets/nessy_app.css`.
- Storybook cleaned `storybook-static`, copied `public/assets`, compiled the manager, then failed compiling the preview.

Representative errors:

- `Error: => Webpack failed, learn more with --debug-webpack`
- Same modern-syntax parse failure class as dev mode, including `??` in dependency code.
- Story-level export breakage also appears:
  - `webpack/doc/review_form/Categories.stories.mdx` imports `productCategories` from `../../assets/javascripts/reviews/store`
  - `webpack/assets/javascripts/reviews/store/index.js` no longer exports `productCategories`
  - `productCategories` now exists in `webpack/assets/javascripts/reviews/store/products.selectors.js`

Cleanup performed after verification:

- Restored `.storybook/preview-head.html` to its pre-build `/assets/nessy_app.css` reference.
- Removed generated `.storybook/assets/` and `storybook-static/`.
- Final UE status returned to only the pre-existing unrelated untracked files:
  - `webpack/assets/javascripts/util/react_mount.js`
  - `webpack/spec/util/react_mount.spec.js`

Browser note:

- I attempted to inspect `http://localhost:3091/` in the in-app browser, but the browser bridge closed while opening the tab. Curl plus Storybook logs are the runtime evidence for this audit.

## Story Inventory

Inventory command included `rg --files -g '*.stories.*'`.

Total story-like files found: 45.

Active root Storybook files loaded by `.storybook/main.js`: 44.

Inactive or not loaded by the active root config: 1.

| Area | Count | Representative paths | Notes |
| --- | ---: | --- | --- |
| Intro | 1 | `webpack/doc/Intro.stories.mdx` | Sidebar intro and framing copy. |
| Base component docs | 1 | `webpack/doc/base/Button.stories.mdx` | Single base Button docs page. |
| Review form Phoenix docs | 21 | `webpack/doc/review_form/*.stories.mdx`, including `EntireReviewForm/Root.stories.mdx`, `EntireShortReviewForm/Root.stories.mdx`, `Categories.stories.mdx`, `TextAnswer.*.stories.mdx`, legal fields, search boxes, footer/header | The main useful active story area. Covers review form states and hard-to-reach question variants. |
| Utility atomics docs | 11 | `webpack/doc/utilities/Colors.stories.mdx`, `Spacing.stories.mdx`, `Flexbox.stories.mdx`, `Border.stories.mdx`, etc. | Legacy utility/atomic class documentation; intro copy now points some users toward Tailwind defaults. |
| Legacy review form JS stories | 8 | `webpack/spec/reviews/components/stories/EntireReviewForm/*.stories.js`, `_QA.stories.js`, `RewardCountryWarning.stories.js` | Older NeRF/legacy review form scenarios. |
| Other composite examples | 2 | `webpack/spec/components/CompanyDomainBanner.stories.js`, `webpack/spec/compare_reports/compare-reports.stories.js` | Isolated component/composite examples. |
| Not loaded | 1 | `spec/components/stories/application.stories.json` | Leftover ViewComponent Storybook JSON path outside active root globs. |

Observed top-level titles:

- `Start Here`
- `Components/Base/Button`
- `Review Form/...`
- `Utilities/...`
- `Components/Composite/...`

## Strategic Use Signal

What Storybook appears intended for:

- NeRF/review-form component development in isolated states.
- Documentation of edge cases that are hard to reproduce in Rails flows.
- Actions addon tracing of event-driven review-form interactions.
- Widget Library iframe access to this Storybook via `engines/widget_library/app/views/widget_library/storybooks/nerf.html.slim`.
- Utility/atomic CSS documentation.

Evidence:

- `.storybook/README.md` describes Storybook as both a development-time aid and documentation tool.
- `webpack/assets/javascripts/reviews/components/README.md` explicitly frames Storybook as the fast way to develop NeRF visual states, edge cases, and factories.
- The Widget Library `nerf.html.slim` points development to `http://localhost:3091` and non-development to `/storybook/`.

Counter-signal:

- The NeRF README still says to run `yarn storybook:nerf`, but `package.json` has no `storybook:nerf` script.
- Root `.storybook/README.md` says "Making a new Storybook" by creating module folders under `.storybook`, but only the root `.storybook` remains.
- Static deployment language says "Not yet", while history shows staging/static work existed later and CI was subsequently removed.

## Maintenance Signal

Positive or neutral:

- Story files still exist and are tied to review-form scenarios.
- Recent commits touched story-related paths:
  - `c2d5d9cba7 2025-11-13 chore: Update most of barlow to figtree (#35746)`
  - `7b9d26b3d7 2026-01-28 feat: Review Form Single Question Layout Experiment [G2AI-246] (#36754)`
- Package dependencies still include Storybook scripts and addons.

Negative:

- Local dev and static build both fail today.
- Storybook dependency line is still on 6.5.x, an old Webpack 4-era setup.
- Current dependency code now includes syntax that this Storybook/Webpack path does not transpile correctly.
- At least one loaded story has a stale import/export contract (`productCategories`).
- `.github/workflows/storybook.yml` was removed in `2352e8e460 ci: Remove storybook workflow`.
- ViewComponent Storybook was intentionally dropped in `7bddd9773a chore: Drop ViewComponent storybook (#33179)`.
- Docs reference removed scripts and old deployment/status assumptions.
- `storybook-addon-recoil-flow` remains in dependencies but is not listed in `.storybook/main.js` addons.

Overall maintenance read: Storybook is still present because historical review-form stories remain useful, but it does not look actively maintained as a reliable local/CI workflow.

## Recommendation

Do not upgrade or fix Storybook inside this audit issue.

Decision recommendation:

- If the review-form team still relies on isolated state development, keep Storybook but treat repair as explicit owned work.
- If there is no current owner or current workflow depending on it, deprecate it and migrate any valuable examples into the supported component documentation path.

Repair path if keeping:

1. Upgrade the Storybook builder path off this Webpack 4-era setup, or explicitly transpile modern dependency packages that now ship `??`, optional chaining, and modern ESM.
2. Fix loaded story imports, starting with `webpack/doc/review_form/Categories.stories.mdx`.
3. Decide whether `storybook-addon-recoil-flow` is needed; remove or wire it.
4. Update docs to replace `yarn storybook:nerf` with `yarn storybook:dev` / `yarn storybook:start`.
5. Reintroduce a minimal CI/static build check if Storybook remains supported.
6. Revisit the Widget Library iframe route so it does not point users to a broken local surface.

Deprecation path if not keeping:

1. Mark the root Storybook scripts/docs as deprecated.
2. Preserve or migrate only the review-form scenarios that still carry product value.
3. Remove stale Widget Library Storybook entry points and obsolete docs in a dedicated cleanup issue.

My practical recommendation: deprecate by default unless a current review-form owner confirms they still want this workflow. If they do, create a focused repair issue before any migration/upgrade work.
