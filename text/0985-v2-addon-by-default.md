---
stage: accepted
start-date: 2023-11-02T16:05:11.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/985
project-link:
suite:
---

# Change the default addon blueprint to `@ember/addon-blueprint`

## Summary

This RFC proposes making [`@ember/addon-blueprint`](https://github.com/ember-cli/ember-addon-blueprint) the default blueprint for new Ember addons, replacing the current v1 and v2 blueprints. The existing blueprints present significant technical challenges that impact developer productivity and ecosystem compatibility. The new blueprint addresses these issues through modern tooling, streamlined architecture, and comprehensive TypeScript integration based on extensive community feedback and production usage patterns.

## Motivation

The current default v1 addon blueprint generates addons that get rebuilt by every consuming app, which is slow and couples addon builds to app builds. There is no built-in path to modern tooling like TypeScript, Glint, or Vite.

`@ember/addon-blueprint` already exists and has been widely adopted by the community. Making it the default gives new addon authors a working setup with single-package structure, Glint for template type safety, native classes and strict mode throughout, and sensible tooling defaults out of the box.

## Detailed design

### Definitions

**V2 Addon**: An addon with `ember-addon.version: 2` in package.json, as defined by [RFC 507](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format/).

**Single-package addon**: An addon with its test suite in the same package, rather than a separate test app in a monorepo.

**Blueprint**: A code generation template used by ember-cli to scaffold projects.

### Blueprint Structure

```
my-addon/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # CI pipeline
│       └── push-dist.yml             # Publish to dist branch
├── src/                              # Source code (published)
│   ├── index.js                      # Main entry point
│   └── template-registry.ts          # Glint type registry
├── tests/                            # Test files
│   ├── index.html                    # Test runner page
│   └── test-helper.ts                # Test setup
├── demo-app/                         # Demo application
│   ├── app.gts                       # Demo app entry point
│   ├── styles.css
│   └── templates/
├── unpublished-development-types/    # Dev-only types
│   └── index.d.ts
├── config/
│   └── ember-cli-update.json         # Blueprint version tracking
├── dist/                             # Built output (gitignored, published)
├── declarations/                     # TS declarations (gitignored, published)
├── package.json
├── index.html                        # Demo app entry page
├── rollup.config.mjs                 # Production build
├── vite.config.mjs                   # Dev build + tests
├── tsconfig.json                     # Dev TypeScript config
├── tsconfig.publish.json             # Publish TypeScript config
├── babel.config.cjs                  # Dev Babel config
├── babel.publish.config.cjs          # Publish Babel config
├── eslint.config.mjs                 # ESLint flat config
├── .prettierrc.mjs                   # Prettier config
├── .prettierignore
├── .template-lintrc.mjs              # Template linting
├── testem.cjs                        # Test runner config
├── .try.mjs                          # Ember version scenarios
├── .editorconfig
├── .env.development
├── .gitignore
├── README.md
├── CONTRIBUTING.md
├── LICENSE.md
└── addon-main.cjs                    # V1 compat shim
```

**Why single-package by default**: Most addons don't need a monorepo. Single-package is simpler to maintain and publish. Advanced users can still set up monorepos.

**Why dual build systems**: Vite gives fast dev rebuilds and HMR. Rollup gives optimized, tree-shaken production output. Tests run entirely through Vite, no webpack or `ember-auto-import` needed.

**Why Glint**: Template type safety via Volar-based TS server plugins. Works for both TypeScript and JavaScript projects.

### Package Configuration

> [!NOTE]
> Blueprints are living artifacts -- the specific file contents shown below will evolve over time. This RFC focuses on the **design goals and architectural rationale** behind the configurations, not the exact file contents. The blueprint repository is the source of truth for current output.

<details><summary>package.json</summary>

```json
{
  "name": "<%= name %>",
  "version": "0.0.0",
  "description": "The default blueprint for Embroider v2 addons.",
  "keywords": ["ember-addon"],
  "repository": "",
  "license": "MIT",
  "author": "",
  "files": [
    "addon-main.cjs",
    "declarations",
    "dist",
    "src"
  ],
  "ember": {
    "edition": "octane"
  },
  "ember-addon": {
    "version": 2,
    "type": "addon",
    "main": "addon-main.cjs"
  },
  "imports": {
    "#src/*": "./src/*"
  },
  "exports": {
    ".": {
      "types": "./declarations/index.d.ts",
      "default": "./dist/index.js"
    },
    "./addon-main.js": "./addon-main.cjs",
    "./*.css": "./dist/*.css",
    "./*": {
      "types": "./declarations/*.d.ts",
      "default": "./dist/*.js"
    }
  }
}
```

`exports` maps consumer-facing imports to the right files (declarations for types, dist for runtime). `imports` with `#src/*` gives tests and the demo app a clean way to import from source without rebuilding -- but can't be used in `src/` itself because Rollup won't transform those imports.

The `files` array includes `src` alongside `dist` and `declarations` so consumers get source-level go-to-definition in their editors.

</details>

<details><summary>Development vs. Production Configs</summary>

The blueprint splits config into dev and publish variants. This is the key architectural pattern throughout:

- **Dev configs** (`babel.config.cjs`, `tsconfig.json`, `vite.config.mjs`) include macro evaluation, compat transforms, Vite/Embroider types, and test infrastructure
- **Publish configs** (`babel.publish.config.cjs`, `tsconfig.publish.json`, `rollup.config.mjs`) use minimal transforms and omit dev-only APIs

This split matters because macros should be evaluated by the consuming app (not baked in at publish time), and `lint:types` against the publish tsconfig catches accidental usage of Vite or Embroider internals in published code.

</details>

<details><summary>Vite Config (vite.config.mjs)</summary>

```javascript
import { defineConfig } from 'vite';
import { extensions, ember, classicEmberSupport } from '@embroider/vite';
import { babel } from '@rollup/plugin-babel';

// For scenario testing
const isCompat = Boolean(process.env.ENABLE_COMPAT_BUILD);

export default defineConfig({
  plugins: [
    ...(isCompat ? [classicEmberSupport()] : []),
    ember(),
    babel({
      babelHelpers: 'inline',
      extensions,
    }),
  ],
  build: {
    rollupOptions: {
      input: {
        tests: 'tests/index.html',
      },
    },
  },
});
```

</details>

<details><summary>Rollup Config (rollup.config.mjs)</summary>

```javascript
import { babel } from '@rollup/plugin-babel';
import { Addon } from '@embroider/addon-dev/rollup';
import { fileURLToPath } from 'node:url';
import { resolve, dirname } from 'node:path';

const addon = new Addon({
  srcDir: 'src',
  destDir: 'dist',
});

const rootDirectory = dirname(fileURLToPath(import.meta.url));
const babelConfig = resolve(rootDirectory, './babel.publish.config.cjs');
const tsConfig = resolve(rootDirectory, './tsconfig.publish.json');

export default {
  output: addon.output(),
  plugins: [
    addon.publicEntrypoints(['**/*.js', 'index.js', 'template-registry.js']),
    addon.appReexports([
      'components/**/*.js',
      'helpers/**/*.js',
      'modifiers/**/*.js',
      'services/**/*.js',
    ]),
    addon.dependencies(),
    babel({
      extensions: ['.js', '.gjs', '.ts', '.gts'],
      babelHelpers: 'bundled',
      configFile: babelConfig,
    }),
    addon.hbs(),
    addon.gjs(),
    // Emit .d.ts declaration files
    addon.declarations(
      'declarations',
      `npx @glint/ember-tsc -- --declaration --project ${tsConfig}`,
    ),
    addon.keepAssets(['**/*.css']),
    addon.clean(),
  ],
};
```

</details>

<details><summary>TypeScript and Glint</summary>

TypeScript is opt-in via `--typescript`.

The blueprint uses two tsconfigs:

**`tsconfig.json`** (dev) -- includes `src/`, `tests/`, `demo-app/`, and `unpublished-development-types/`. Has Vite and Embroider types so your editor works.

```json
{
  "extends": "@ember/app-tsconfig",
  "include": [
    "src/**/*",
    "tests/**/*",
    "unpublished-development-types/**/*",
    "demo-app/**/*"
  ],
  "compilerOptions": {
    "rootDir": ".",
    "types": [
      "ember-source/types",
      "vite/client",
      "@embroider/core/virtual",
      "@glint/ember-tsc/types"
    ]
  }
}
```

**`tsconfig.publish.json`** -- only `src/` and dev types. No Vite or Embroider types, so `lint:types` catches accidental usage of dev-only APIs in published code.

```json
{
  "extends": "@ember/library-tsconfig",
  "include": ["./src/**/*", "./unpublished-development-types/**/*"],
  "compilerOptions": {
    "allowJs": true,
    "declarationDir": "declarations",
    "rootDir": "./src",
    "types": ["ember-source/types", "@glint/ember-tsc/types"]
  }
}
```

The Glint template registry (`src/template-registry.ts`) lets apps using loose mode (hbs files) consume your types. Not needed if your library only targets strict mode consumers.

</details>

<details><summary>The Strict Resolver and <code>modules</code></summary>

Both the test helper and the demo app use `ember-strict-application-resolver` instead of the classic Ember resolver. Instead of filesystem conventions, you explicitly register modules via a `modules` object. Each key must match a `./[type]/[name]` pattern (see [RFC 1132](https://rfcs.emberjs.com/id/1132-default-strict-resolver)):

```typescript
class MyApp extends EmberApp {
  modules = {
    './router': Router,                              // direct assignment
    './services/page-title': PageTitleService,        // explicit import
    ...import.meta.glob('./services/**/*', { eager: true }),  // bulk registration
    ...import.meta.glob('./templates/**/*', { eager: true }),
  };
}
```

You can register modules individually (useful for things from dependencies) or use `import.meta.glob` to sweep up everything in a directory. The glob approach is convenient but imports everything matching the pattern -- if you have non-service files in `services/`, they'll get pulled in too.

This pattern is used in two places:

- **Test helper** -- registers a minimal Router and optionally any services needed for tests
- **Demo app** -- registers the Router, templates, services, and anything else the demo needs

</details>

<details><summary>Demo App</summary>

The blueprint includes a small demo application for manually testing your addon during development. Run `npm start` (or `pnpm start`) to launch Vite's dev server, which serves the root `index.html`:

**`index.html`**:
```html
<!doctype html>
<html lang="en-us">
<head>
  <meta charset="utf-8" />
  <title>Demo App</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="stylesheet" href="./demo-app/styles.css" />
</head>
<body>
  <script type="module">
    import { App } from './demo-app/app';
    App.create({})
  </script>
</body>
</html>
```

**`demo-app/app.gts`**:
```typescript
import EmberApp from 'ember-strict-application-resolver';
import EmberRouter from '@ember/routing/router';
import PageTitleService from 'ember-page-title/services/page-title';

class Router extends EmberRouter {
  location = 'history';
  rootURL = '/';
}

export class App extends EmberApp {
  modules = {
    './router': Router,
    './services/page-title': PageTitleService,
    ...import.meta.glob('./services/**/*', { eager: true }),
    ...import.meta.glob('./templates/**/*', { eager: true }),
  };
}

Router.map(function () {});
```

The demo app is a real Ember app -- it has routes, templates, and services -- but it boots directly via `ember-strict-application-resolver` with no ember-cli build step. Any addon code you want to exercise in the demo needs to be imported in the demo app's templates or registered in `modules`. The demo app's files are not published (they're not in the `files` array or `exports`).

</details>

<details><summary>Testing</summary>

Tests also run entirely on Vite -- no ember-cli build pipeline, no webpack.

**`tests/test-helper.ts`**:
```typescript
import EmberApp from 'ember-strict-application-resolver';
import EmberRouter from '@ember/routing/router';
import * as QUnit from 'qunit';
import { setApplication } from '@ember/test-helpers';
import { setup } from 'qunit-dom';
import { start as qunitStart, setupEmberOnerrorValidation } from 'ember-qunit';
import { setTesting } from '@embroider/macros';

class Router extends EmberRouter {
  location = 'none';
  rootURL = '/';
}

class TestApp extends EmberApp {
  modules = {
    './router': Router,
    // add any custom services here
    // import.meta.glob('./services/*', { eager: true }),
  };
}

Router.map(function () {});

export function start() {
  setTesting(true);
  setApplication(
    TestApp.create({
      autoboot: false,
      rootElement: '#ember-testing',
    }),
  );
  setup(QUnit.assert);
  setupEmberOnerrorValidation();
  qunitStart();
}
```

**`tests/index.html`**:
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title><%= name %> Tests</title>
    <meta name="description" content="" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body>
    <div id="qunit"></div>
    <div id="qunit-fixture">
      <div id="ember-testing-container">
        <div id="ember-testing"></div>
      </div>
    </div>

    <script src="/testem.js" integrity="" data-embroider-ignore></script>
    <script type="module">
      import "ember-testing";
    </script>

    <script type="module">
      import { start } from "./test-helper.js";
      import.meta.glob("./**/*.{js,ts,gjs,gts}", { eager: true });
      start();
    </script>
  </body>
</html>
```

The test app is structurally the same as the demo app -- a minimal Ember app via `ember-strict-application-resolver` -- but configured for testing (`location = 'none'`, `autoboot: false`, `setTesting(true)`). Test discovery uses `import.meta.glob` in the HTML entry point. This is also a proof-of-concept for how future compat-less Ember apps could work.

#### Cross-Version Testing

The `.try.mjs` config defines scenarios for testing against multiple Ember versions:

```javascript
const compatFiles = {
  'ember-cli-build.cjs': `const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const { compatBuild } = require('@embroider/compat');
module.exports = async function (defaults) {
  const { buildOnce } = await import('@embroider/vite');
  let app = new EmberApp(defaults);
  return compatBuild(app, buildOnce);
};`,
  'config/optional-features.json': JSON.stringify({
    'application-template-wrapper': false,
    'default-async-observers': true,
    'jquery-integration': false,
    'template-only-glimmer-components': true,
    'no-implicit-route-model': true,
  }),
};

const compatDeps = {
  '@embroider/compat': '^4.0.3',
  'ember-cli': '^5.12.0',
  'ember-auto-import': '^2.10.0',
  '@ember/optional-features': '^2.2.0',
};

export default {
  scenarios: [
    {
      name: 'ember-lts-5.8',
      npm: {
        devDependencies: {
          'ember-source': '~5.8.0',
          ...compatDeps,
        },
      },
      env: {
        ENABLE_COMPAT_BUILD: true,
      },
      files: compatFiles,
    },
    {
      name: 'ember-lts-5.12',
      npm: {
        devDependencies: {
          'ember-source': '~5.12.0',
          ...compatDeps,
        },
      },
      env: {
        ENABLE_COMPAT_BUILD: true,
      },
      files: compatFiles,
    },
    {
      name: 'ember-lts-6.4',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@~6.4.0',
        },
      },
    },
    {
      name: 'ember-latest',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@latest',
        },
      },
    },
    {
      name: 'ember-beta',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@beta',
        },
      },
    },
    {
      name: 'ember-alpha',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@alpha',
        },
      },
    },
  ],
};
```

Older Ember versions (5.x) need `@embroider/compat` and an `ember-cli-build.cjs` shim. Ember 6.4+ runs natively without compat mode.

</details>

<details><summary>Babel Configs</summary>

**Dev (`babel.config.cjs`)** -- used by your editor and tests. Includes macro evaluation and compat transforms:

```javascript
/**
 * This babel.config is not used for publishing.
 * It's only for the local editing experience
 * (and linting)
 */
const { buildMacros } = require('@embroider/macros/babel');
const { babelCompatSupport, templateCompatSupport } = require('@embroider/compat/babel');

const macros = buildMacros();

// For scenario testing
const isCompat = Boolean(process.env.ENABLE_COMPAT_BUILD);

module.exports = {
  plugins: [
    ['@babel/plugin-transform-typescript', {
      allExtensions: true,
      allowDeclareFields: true,
      onlyRemoveTypeImports: true,
    }],
    ['babel-plugin-ember-template-compilation', {
      transforms: [
        ...(isCompat ? templateCompatSupport() : macros.templateMacros),
      ],
    }],
    ['module:decorator-transforms', {
      runtime: {
        import: require.resolve('decorator-transforms/runtime-esm'),
      },
    }],
    ...(isCompat ? babelCompatSupport() : macros.babelMacros),
  ],
  generatorOpts: {
    compact: false,
  },
};
```

**Publish (`babel.publish.config.cjs`)** -- minimal transforms. Macros are deliberately omitted; the consuming app evaluates them:

```javascript
/**
 * This babel.config is only used for publishing.
 *
 * For local dev experience, see the babel.config
 */
module.exports = {
  plugins: [
    ['@babel/plugin-transform-typescript', {
      allExtensions: true,
      allowDeclareFields: true,
      onlyRemoveTypeImports: true,
    }],
    ['babel-plugin-ember-template-compilation', {
      targetFormat: 'hbs',
      transforms: [],
    }],
    ['module:decorator-transforms', {
      runtime: {
        import: 'decorator-transforms/runtime-esm',
      },
    }],
  ],
  generatorOpts: {
    compact: false,
  },
};
```

</details>

<details><summary>Linting</summary>

**ESLint** uses flat config (`eslint.config.mjs`) with `defineConfig` and `globalIgnores`:

```javascript
import babelParser from '@babel/eslint-parser/experimental-worker';
import js from '@eslint/js';
import { defineConfig, globalIgnores } from 'eslint/config';
import prettier from 'eslint-config-prettier';
import ember from 'eslint-plugin-ember/recommended';
import importPlugin from 'eslint-plugin-import';
import n from 'eslint-plugin-n';
import globals from 'globals';
import ts from 'typescript-eslint';

const esmParserOptions = {
  ecmaFeatures: { modules: true },
  ecmaVersion: 'latest',
};

const tsParserOptions = {
  projectService: true,
  tsconfigRootDir: import.meta.dirname,
};

export default defineConfig([
  globalIgnores(['dist/', 'dist-*/', 'declarations/', 'coverage/', '!**/.*']),
  js.configs.recommended,
  prettier,
  ember.configs.base,
  ember.configs.gjs,
  ember.configs.gts,
  {
    linterOptions: {
      reportUnusedDisableDirectives: 'error',
    },
  },
  {
    files: ['**/*.js'],
    languageOptions: {
      parser: babelParser,
    },
  },
  {
    files: ['**/*.{js,gjs}'],
    languageOptions: {
      parserOptions: esmParserOptions,
      globals: { ...globals.browser },
    },
  },
  {
    files: ['**/*.{ts,gts}'],
    languageOptions: {
      parser: ember.parser,
      parserOptions: tsParserOptions,
      globals: { ...globals.browser },
    },
    extends: [
      ...ts.configs.recommendedTypeChecked,
      { ...ts.configs.eslintRecommended, files: undefined },
      ember.configs.gts,
    ],
  },
  {
    files: ['src/**/*'],
    plugins: { import: importPlugin },
    rules: {
      'import/extensions': ['error', 'always', { ignorePackages: true }],
    },
  },
  // ... additional CJS/ESM node file configurations
]);
```

Type-aware linting via `projectService`, Ember .gjs/.gts support, and enforced file extensions in `src/` imports.

**Template linting** (`.template-lintrc.mjs`):
```javascript
export default {
  extends: 'recommended',
  checkHbsTemplateLiterals: false,
};
```

**Prettier** (`.prettierrc.mjs`):
```javascript
export default {
  plugins: ['prettier-plugin-ember-template-tag'],
  overrides: [
    {
      files: '*.{js,gjs,ts,gts,mjs,mts,cjs,cts}',
      options: {
        singleQuote: true,
        templateSingleQuote: false,
      },
    },
  ],
};
```

</details>

<details><summary>CI/CD</summary>

The blueprint generates GitHub Actions workflows (shown here with pnpm; npm/yarn variants are also generated):

**CI workflow** (`ci.yml`):
- **Lint** -- runs `pnpm lint` (ESLint, Prettier, template-lint, type checking)
- **Test** -- runs `pnpm test`, then outputs a scenario matrix via `@embroider/try list`
- **Floating deps** -- installs without lockfile, runs tests to catch compatibility issues early
- **Try scenarios** -- matrix job that applies each `.try.mjs` scenario and runs tests against it

```yaml
try-scenarios:
  name: ${{ matrix.name }}
  runs-on: ubuntu-latest
  needs: "test"
  timeout-minutes: 10
  strategy:
    fail-fast: false
    matrix: ${{fromJson(needs.test.outputs.matrix)}}
  steps:
    - name: Apply Scenario
      run: pnpm dlx @embroider/try apply ${{ matrix.name }}
    - name: Install Dependencies
      run: pnpm install --no-lockfile
    - name: Run Tests
      run: pnpm test
      env: ${{ matrix.env }}
```

**Push dist workflow** (`push-dist.yml`) -- on push to main, builds the addon and pushes compiled assets to a `dist` branch for git-based consumption.

</details>

<details><summary>V1 Compatibility</summary>

```javascript
// addon-main.cjs
'use strict';
const { addonV1Shim } = require('@embroider/addon-shim');
module.exports = addonV1Shim(__dirname);
```

This shim translates V2 package metadata into V1 build hooks so the addon works in classic ember-cli apps.

</details>

### Influence on Future App Blueprint

The test setup here is a proof-of-concept for future compat-less Ember apps:

- A minimal Ember app running on Vite with no webpack or `ember-cli-build.js`
- Bootstrap with just `EmberApp` and `EmberRouter` -- no complex build pipeline
- ES modules and `import.meta.glob` for module discovery instead of AMD/requirejs
- Direct framework API usage instead of ember-cli abstractions

This validates that Ember apps can run well on modern build tools, pointing toward simpler app blueprints in the future.

### Migration

Existing addons are unaffected. New addons get the new blueprint automatically. Existing addons can migrate by generating a new project and copying relevant files, or using `npx ember-cli@latest addon <name> --blueprint @ember/addon-blueprint`.

#### `ember-cli-update` Support

The blueprint includes `config/ember-cli-update.json` so that `ember-cli-update` continues to work. This file tracks the blueprint package name and version, allowing `ember-cli-update` to detect available updates and apply them. The entry should reference `@ember/addon-blueprint` and the version used to generate the addon, following the same pattern used by the app blueprint.

#### Codemod

A codemod for migrating existing v1 addons to the new blueprint structure is out of scope for this RFC, but would be a valuable follow-up effort. [Mainmatter](https://mainmatter.com/) has expressed interest in developing such a codemod. In the meantime, addon authors can generate a fresh project with the new blueprint and manually move their source code into it. The [embroider-build/embroider](https://github.com/embroider-build/embroider) repo also has documentation on how to work with and migrate to v2 addons.

## How we teach this

### Documentation Updates

- Update the Ember Guides and CLI docs to reference the new blueprint
- The blueprint README covers customization, publishing, and multi-version support
- Provide migration guides for v1 and v2 addon authors
- The blueprint should generate parallel `.md` files (or inline comments) alongside config files to explain the purpose and rationale of each configuration. This helps addon authors understand *why* a config exists, not just *what* it contains, and reduces confusion when configs change across blueprint versions

### Key Concepts for Addon Authors

#### `exports` and `imports`

`exports` defines your addon's public API:

```json
{
  "exports": {
    ".": {
      "types": "./declarations/index.d.ts",
      "default": "./dist/index.js"
    },
    "./*.css": "./dist/*.css",
    "./*": {
      "types": "./declarations/*.d.ts",
      "default": "./dist/*.js"
    }
  }
}
```

`imports` with `#src/*` lets tests and the demo app import from source without rebuilding. Can't be used in `src/` (Rollup won't transform these). Files in `src/` must use relative imports.

#### Importing Addon Code in Tests: `#src/*` vs. Consumer-Style

When writing tests, you have two ways to import from your addon:

**`#src/*` imports** -- import directly from source files:
```javascript
import { myHelper } from '#src/helpers/my-helper';
```
- Works immediately, no build step needed
- Fast feedback loop during development
- Tests the source code directly

**Consumer-style imports** -- import as a consumer would:
```javascript
import { myHelper } from 'my-addon/helpers/my-helper';
```
- Tests the published API surface
- Requires `dist/` to exist (needs a build first)
- Catches issues with `exports` mapping or build transforms

**Recommendation**: Use `#src/*` imports for day-to-day development. The try-scenarios CI matrix will catch build/export issues by running against real builds. If you need to specifically test the published output, use consumer-style imports in a dedicated test file and run `npm run build` first.

#### Self-Imports

Self-imports (e.g. `import { x } from 'my-addon/foo'`) don't work during development in `src/` files because they resolve through `exports` to `dist/`, which doesn't exist until you build. Use relative imports in `src/`:

```javascript
// In src/ files:
import { myHelper } from './helpers/my-helper'; // yes
import { myHelper } from 'my-addon/helpers/my-helper'; // no
```

#### Dev vs. Publish Configs

| Purpose | Dev | Publish |
|---------|-----|---------|
| Babel | `babel.config.cjs` (macros, compat) | `babel.publish.config.cjs` (minimal) |
| TypeScript | `tsconfig.json` (all files, Vite types) | `tsconfig.publish.json` (src only) |
| Build | `vite.config.mjs` (HMR, tests) | `rollup.config.mjs` (tree-shaking) |

Macros are evaluated in dev for testing but left unevaluated in published output -- the consuming app handles them. The publish tsconfig omits Vite/Embroider types so `lint:types` catches accidental usage.

#### Monorepo Setup

The single-package default works for most addons. If you need a monorepo (complex integration testing, multiple related packages, full documentation app):

1. Generate addon: `npx ember-cli@latest addon my-addon --blueprint @ember/addon-blueprint --skip-git`
2. Remove generated test infrastructure
3. Generate test app: `npx ember-cli@latest app test-app --blueprint @ember/app-blueprint`
4. Set up workspace tooling (pnpm/yarn workspaces)
5. Install addon in test app

#### Unpublished Addons in a Monorepo

Sometimes you have a v2 addon in a monorepo that's only consumed by other packages in the workspace -- it's never published to npm. In this case you can skip the build step entirely and point `exports` at your source files:

```json
{
  "name": "my-internal-addon",
  "ember-addon": {
    "version": 2,
    "type": "addon",
    "main": "addon-main.cjs"
  },
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./*": {
      "types": "./src/*.ts",
      "default": "./src/*.ts"
    }
  }
}
```

Key differences from a published addon:
- `exports` points to `src/` instead of `dist/` and `declarations/`
- No `files` array needed (not publishing to npm)
- No rollup build, no `prepack` script, no `declarations/` directory
- No `babel.publish.config.cjs` or `tsconfig.publish.json` needed
- You still need `addon-main.cjs` if any consuming app in the workspace uses the classic ember-cli build

The consuming app's build tooling (Vite/Embroider) handles the transpilation. This is much simpler to maintain for workspace-internal code.

#### Publishing

1. Write code in `src/`, tests with `#src/*` imports
2. `npm run build` runs Rollup with publish configs, producing `dist/` and `declarations/`
3. `npm publish` ships only `files` from package.json
4. Consumers import via `exports`, not internal paths

### Resources

- [@ember/addon-blueprint README](https://github.com/emberjs/ember-addon-blueprint#readme)
- [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md)
- [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)
- [Node.js Package Exports](https://nodejs.org/api/packages.html#exports)
- [Glint Documentation](https://typed-ember.gitbook.io/glint/)

## Drawbacks

- Some advanced use cases (monorepos, custom builds) need additional configuration.
- Addon authors unfamiliar with TypeScript/Glint face a learning curve, but JavaScript is fully supported.
- The blueprint is opinionated, but covers the vast majority of use cases.

## Alternatives

- Do nothing -- this should have shipped years ago. The community has already broadly adopted v2 addons as the de facto default; the official defaults are lagging behind actual community practice.
- Default to monorepo (too complex for most users (and maintainers of the bluleprints, as it turns out))
- Provide multiple blueprints (maintenance burden, confusion)
  - this is slightly addressed by documenting how to compose blueprints for differentt workflows, like having multiple test apps, for example.

## Unresolved questions

- How to best support advanced monorepo setups in the future.

## Previously unresolved questions

### V1 vs V2 format Usage?

Data accumulated from Apr 2024 -> Mar 2026


Top 100 addons per category, 
- seprated into years by last published, 
- sorted by the number of downloads in the last month (as of 2026-04-07).

> [!NOTE]
> In the outdated section, some of the addons listed *can* work and/or have purpose in with modern (and compatless) build tools, such as vite, but they'll need to be updated first in order to do so (@sentry/ember, for example, has a PR to convert to a v2 addon -- and ember-a11y-testing _recently_ was converted to v2 addon + a testem middleware)

Tho also note that many old addons simply are not needed anymore (polyfills, vite-native behavior, or vite-plugins). This situations will be noted in that table below.


<details>
<summary>

#### Top 100 V2 Addons (Apr 2024 - Mar 2026)

</summary>

Last published breakdown: 

2022: 2, 
2023: 2, 
2024: 8, 
2025: 50, 
2026: 38

**2022**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| @ember-compat/tracked-built-ins | 10,032,255 | 114,640 | 2022-11-16 | 0.9.1 |
| ember-css-url | 7,836,462 | 83,189 | 2022-07-06 | 1.0.0 |

**2023**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-lifeline | 23,859,567 | 174,334 | 2023-05-05 | 7.0.0 |
| ember-route-template | 14,121,594 | 151,868 | 2023-11-13 | 1.0.3 |

**2024**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| @glimmer/component | 120,678,687 | 891,505 | 2024-10-29 | 2.0.0 |
| ember-cookies | 43,200,441 | 266,028 | 2024-12-28 | 1.3.0 |
| ember-sinon-qunit | 28,788,417 | 227,878 | 2024-07-17 | 7.5.0 |
| ember-window-mock | 24,875,874 | 195,958 | 2024-08-30 | 1.0.2 |
| ember-click-outside | 23,909,796 | 120,402 | 2024-10-05 | 6.1.1 |
| ember-autofocus-modifier | 12,824,406 | 83,872 | 2024-01-17 | 7.0.1 |
| ember-browser-services | 5,370,219 | 42,163 | 2024-10-25 | 5.0.1 |
| ember-phone-input | 829,086 | 14,193 | 2024-01-22 | 10.0.0 |

**2025**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| @ember/test-helpers | 156,315,537 | 986,217 | 2025-11-14 | 5.4.1 |
| ember-qunit | 144,226,467 | 884,380 | 2025-09-12 | 9.0.4 |
| @ember/test-waiters | 124,979,085 | 822,155 | 2025-07-05 | 4.1.1 |
| ember-truth-helpers | 112,821,660 | 670,990 | 2025-09-12 | 5.0.0 |
| ember-inflector | 105,370,749 | 639,229 | 2025-03-12 | 6.0.0 |
| ember-basic-dropdown | 75,006,828 | 486,050 | 2025-12-14 | 8.11.0 |
| ember-element-helper | 72,573,876 | 464,705 | 2025-05-12 | 0.8.8 |
| ember-page-title | 68,902,623 | 417,690 | 2025-08-22 | 9.0.3 |
| ember-assign-helper | 59,464,890 | 393,324 | 2025-05-12 | 0.5.1 |
| ember-style-modifier | 65,645,721 | 380,179 | 2025-08-25 | 4.5.1 |
| ember-power-select | 55,148,391 | 343,062 | 2025-12-07 | 8.12.1 |
| ember-cli-string-helpers | 39,804,336 | 272,365 | 2025-07-22 | 8.0.1 |
| ember-validators | 44,187,849 | 269,841 | 2025-04-21 | 5.0.0 |
| ember-resources | 25,250,382 | 216,372 | 2025-08-23 | 7.0.7 |
| ember-math-helpers | 25,403,769 | 213,128 | 2025-04-07 | 5.0.0 |
| ember-cli-page-object | 27,238,554 | 211,158 | 2025-09-05 | 2.3.2 |
| ember-raf-scheduler | 39,835,962 | 206,235 | 2025-06-06 | 0.5.0 |
| @embroider/router | 20,936,853 | 200,194 | 2025-11-25 | 3.0.6 |
| ember-async-data | 24,012,477 | 180,617 | 2025-05-12 | 2.0.1 |
| ember-keyboard | 23,456,817 | 166,736 | 2025-10-03 | 9.0.4 |
| ember-set-helper | 20,683,305 | 161,333 | 2025-10-04 | 3.1.0 |
| ember-can | 19,178,721 | 145,220 | 2025-04-15 | 8.0.0 |
| ember-power-calendar | 26,722,791 | 136,768 | 2025-09-16 | 1.8.0 |
| liquid-fire | 19,685,034 | 123,759 | 2025-05-13 | 0.37.1 |
| ember-changeset | 27,685,728 | 123,396 | 2025-05-01 | 5.0.0 |
| ember-mirage | 5,529,483 | 119,505 | 2025-06-26 | 0.4.3 |
| ember-stargate | 9,450,171 | 107,922 | 2025-09-06 | 1.0.2 |
| ember-sortable | 24,548,643 | 107,884 | 2025-08-13 | 5.3.3 |
| ember-curry-component | 7,463,016 | 107,418 | 2025-08-21 | 0.3.1 |
| ember-changeset-validations | 18,984,825 | 102,566 | 2025-05-01 | 5.0.0 |
| @nullvoxpopuli/legacy-prototype-extensions | 2,516,580 | 91,185 | 2025-06-06 | 0.1.0 |
| @docfy/ember | 6,407,964 | 85,641 | 2025-12-08 | 0.11.0 |
| ember-power-calendar-moment | 16,037,172 | 83,493 | 2025-05-12 | 1.0.4 |
| ember-modify-based-class-resource | 5,928,381 | 74,244 | 2025-05-10 | 1.1.2 |
| ember-link | 11,872,053 | 70,489 | 2025-08-07 | 3.5.0 |
| ember-data-resources | 6,219,954 | 69,616 | 2025-12-23 | 5.3.2 |
| ember-animated | 14,764,860 | 67,917 | 2025-05-17 | 2.2.0 |
| @fortawesome/ember-fontawesome | 8,417,979 | 67,650 | 2025-08-19 | 3.1.0 |
| ember-router-helpers | 9,176,418 | 67,601 | 2025-12-08 | 1.0.1 |
| ember-render-helpers | 10,498,860 | 66,065 | 2025-09-12 | 2.0.0 |
| @embroider/legacy-inspector-support | 2,260,512 | 65,748 | 2025-10-07 | 0.1.3 |
| ember-drag-drop | 7,950,051 | 64,952 | 2025-02-18 | 1.0.1 |
| ember-infinity | 8,708,130 | 64,831 | 2025-04-29 | 3.0.2 |
| ember-css-transitions | 8,554,581 | 63,380 | 2025-04-15 | 4.5.0 |
| reactiveweb | 6,702,777 | 60,764 | 2025-12-23 | 1.9.1 |
| ember-flatpickr | 7,872,498 | 48,692 | 2025-11-10 | 9.0.2 |
| ember-power-select-with-create | 9,333,036 | 44,163 | 2025-11-07 | 3.1.0 |
| ember-cli-notifications | 4,247,838 | 42,987 | 2025-01-08 | 9.1.0 |
| ember-pikaday | 695,939 | 31,145 | 2025-02-05 | 5.1.1 |
| ember-promise-modals | 808,287 | 14,750 | 2025-07-16 | 5.0.1 |

**2026**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-source | 186,808,986 | 1,252,091 | 2026-03-31 | 6.12.0 |
| ember-modifier | 137,285,181 | 866,324 | 2026-02-03 | 4.3.0 |
| ember-concurrency | 110,864,934 | 664,342 | 2026-02-18 | 5.2.0 |
| ember-data | 95,194,035 | 539,691 | 2026-01-12 | 5.8.1 |
| ember-cli-deprecation-workflow | 64,106,181 | 478,222 | 2026-02-27 | 4.0.1 |
| @ember-data/model | 86,939,136 | 462,460 | 2026-01-12 | 5.8.1 |
| @ember-data/store | 85,860,918 | 459,409 | 2026-01-12 | 5.8.1 |
| @ember-data/adapter | 84,414,753 | 459,045 | 2026-01-12 | 5.8.1 |
| @ember-data/serializer | 82,333,215 | 455,863 | 2026-01-12 | 5.8.1 |
| @ember-data/debug | 80,805,258 | 443,057 | 2026-01-12 | 5.8.1 |
| tracked-built-ins | 58,917,069 | 378,673 | 2026-03-20 | 4.1.2 |
| ember-intl | 45,622,008 | 265,339 | 2026-04-01 | 8.2.2 |
| ember-simple-auth | 34,333,533 | 227,886 | 2026-02-02 | 8.3.0 |
| @ember-data/request | 37,042,326 | 207,429 | 2026-01-12 | 5.8.1 |
| ember-moment | 31,260,276 | 204,677 | 2026-03-25 | 11.1.0 |
| @ember-data/legacy-compat | 36,155,826 | 196,895 | 2026-01-12 | 5.8.1 |
| @ember-data/json-api | 36,104,535 | 196,238 | 2026-01-12 | 5.8.1 |
| @ember-data/graph | 35,991,936 | 193,469 | 2026-01-12 | 5.8.1 |
| ember-focus-trap | 26,904,186 | 188,952 | 2026-03-17 | 2.0.0 |
| @warp-drive/core-types | 22,726,737 | 187,067 | 2026-01-12 | 5.8.1 |
| @html-next/vertical-collection | 27,052,803 | 172,944 | 2026-02-27 | 5.0.3 |
| ember-cli-flash | 25,021,728 | 170,176 | 2026-02-10 | 7.0.0 |
| ember-a11y-testing | 24,872,517 | 158,520 | 2026-02-19 | 8.0.0 |
| @nullvoxpopuli/ember-composable-helpers | 10,309,275 | 139,480 | 2026-03-21 | 5.3.1 |
| @ember-data/request-utils | 23,063,526 | 130,831 | 2026-01-12 | 5.8.1 |
| ember-a11y-refocus | 9,429,003 | 106,948 | 2026-02-26 | 5.2.1 |
| tracked-toolbox | 17,698,059 | 105,020 | 2026-01-09 | 2.2.0 |
| ember-file-upload | 21,576,303 | 102,266 | 2026-04-03 | 10.0.0 |
| @hashicorp/design-system-components | 8,326,476 | 101,277 | 2026-03-09 | 6.1.0 |
| @warp-drive/core | 4,957,776 | 88,635 | 2026-01-12 | 5.8.1 |
| @warp-drive/utilities | 4,858,560 | 87,396 | 2026-01-12 | 5.8.1 |
| @warp-drive/legacy | 4,144,599 | 71,998 | 2026-01-12 | 5.8.1 |
| @warp-drive/json-api | 4,051,980 | 71,309 | 2026-01-12 | 5.8.1 |
| ember-launch-darkly | 9,265,005 | 69,901 | 2026-03-12 | 6.0.1 |
| ember-welcome-page | 12,674,124 | 66,339 | 2026-01-10 | 8.0.5 |
| @warp-drive/ember | 5,977,287 | 65,540 | 2026-01-12 | 5.8.1 |
| ember-provide-consume-context | 4,746,339 | 49,158 | 2026-02-13 | 0.9.0 |
| ember-primitives | 3,567,699 | 41,338 | 2026-04-04 | 0.55.1 |

</details>

<details>
<summary>

#### Top 100 V1 Addons (Apr 2024 - Mar 2026)

</summary>

Last published breakdown: 
2017: 1, 
2018: 4, 
2019: 7, 
2020: 15, 
2021: 10, 
2022: 15, 
2023: 11, 
2024: 9, 
2025: 23, 
2026: 5

**2017**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-ignore-children-helper | 5,625,351 | 42,238 | 2017-10-17 | 1.0.1 |

**2018**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-cli-blueprint-test-helpers | 3,867,003 | 25,945 | 2018-12-18 | 0.19.2 |
| ember-hammertime | 379,223 | 23,432 | 2018-10-16 | 1.6.0 |
| ember-element-resize-detector | 446,800 | 19,411 | 2018-12-03 | 0.4.0 |
| ember-power-select-blockless | 874,116 | 1,990 | 2018-05-19 | 0.5.0 |

**2019**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-radio-button | 11,871,756 | 65,725 | 2019-05-01 | 2.0.1 |
| ember-cli-html-minifier | 1,294,164 | 65,327 | 2019-12-12 | 1.1.0 |
| ember-cli-document-title-northm | 381,290 | 29,684 | 2019-04-15 | 1.0.3 |
| @ember-decorators/babel-transforms | 400,309 | 21,271 | 2019-04-06 | 5.2.0 |
| ember-cli-yuidoc | 3,017,952 | 21,191 | 2019-11-02 | 0.9.1 |
| ember-cli-github-pages | 409,371 | 18,512 | 2019-06-13 | 0.2.2 |
| ember-i18n | 396,083 | 14,663 | 2019-01-29 | 5.3.1 |

**2020**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-copy | 36,201,195 | 227,392 | 2020-10-12 | 2.0.1 |
| @ember/ordered-set | 23,710,779 | 159,105 | 2020-10-28 | 4.0.0 |
| ember-text-measurer | 38,523,951 | 156,431 | 2020-05-22 | 0.6.0 |
| ember-event-helpers | 15,972,858 | 58,957 | 2020-04-07 | 0.1.1 |
| ember-cli-chart | 369,193 | 42,441 | 2020-08-06 | 3.7.2 |
| ember-gestures | 478,722 | 39,087 | 2020-08-16 | 2.0.1 |
| ember-api-actions | 610,729 | 36,082 | 2020-04-04 | 0.2.9 |
| ember-diff-attrs | 664,955 | 31,268 | 2020-10-02 | 0.2.3 |
| ember-on-modifier | 439,919 | 25,394 | 2020-04-16 | 1.0.1 |
| ember-data-url-templates | 402,510 | 20,550 | 2020-08-03 | 0.4.5 |
| ember-web-app | 447,920 | 18,568 | 2020-10-30 | 5.0.1 |
| ember-data-change-tracker | 384,368 | 18,232 | 2020-04-28 | 0.10.1 |
| ember-scrollable | 401,706 | 15,521 | 2020-02-06 | 1.0.2 |
| ember-light-table | 385,517 | 15,499 | 2020-07-29 | 3.0.0-beta.0 |
| ember-set-body-class | 951,840 | 11,303 | 2020-12-02 | 1.0.2 |

**2021**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-inline-svg | 13,562,199 | 114,939 | 2021-11-23 | 1.0.1 |
| ember-promise-helpers | 9,307,881 | 58,694 | 2021-11-09 | 2.0.0 |
| ember-router-scroll | 8,886,861 | 43,524 | 2021-11-18 | 4.1.2 |
| ember-cli-less | 665,738 | 39,042 | 2021-06-17 | 3.0.2 |
| ember-deep-set | 863,032 | 27,954 | 2021-12-05 | 0.3.0 |
| ember-cli-accounting | 551,793 | 24,502 | 2021-12-30 | 2.1.0 |
| ember-cli-document-title | 397,451 | 15,312 | 2021-01-09 | 1.1.0 |
| qunit-console-grouper | 591,548 | 8,708 | 2021-02-03 | 0.3.0 |
| ember-steps | 466,940 | 2,194 | 2021-05-26 | 10.0.1 |
| ember-swiper6 | 394,460 | 69 | 2021-08-08 | 2.1.6 |

**2022**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-in-viewport | 26,149,644 | 213,632 | 2022-11-09 | 4.1.0 |
| ember-buffered-proxy | 11,741,013 | 131,064 | 2022-05-11 | 2.1.1 |
| ember-short-number | 4,172,391 | 64,615 | 2022-04-30 | 2.0.0 |
| active-model-adapter | 17,671,680 | 55,611 | 2022-10-23 | 4.0.1 |
| ember-resize-observer-service | 14,524,146 | 46,977 | 2022-05-28 | 1.1.0 |
| ember-app-scheduler | 8,062,362 | 43,661 | 2022-01-29 | 7.0.1 |
| ember-toggle | 5,949,477 | 39,883 | 2022-02-02 | 9.0.3 |
| ember-angle-bracket-invocation-polyfill | 813,456 | 38,859 | 2022-05-18 | 3.0.2 |
| ember-tag-input | 548,162 | 28,288 | 2022-10-30 | 3.1.0 |
| ember-component-css | 592,908 | 27,841 | 2022-05-09 | 0.8.1 |
| ember-cli-build-notifications | 521,898 | 27,807 | 2022-12-30 | 2.0.0 |
| ember-websockets | 354,380 | 16,125 | 2022-05-23 | 10.2.1 |
| ember-useragent | 477,428 | 15,226 | 2022-03-01 | 0.12.0 |
| ember-cli-browser-navigation-button-test-helper | 627,044 | 10,126 | 2022-02-26 | 0.3.0 |
| ember-cropperjs | 462,722 | 4,920 | 2022-12-15 | 0.9.6 |

**2023**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| @ember-data/canary-features | 50,459,823 | 260,167 | 2023-02-27 | 4.11.3 |
| @ember-data/record-data | 46,618,128 | 250,195 | 2023-02-27 | 4.11.3 |
| ember-table | 10,029,222 | 48,353 | 2023-07-30 | 5.0.6 |
| ember-arg-types | 8,765,289 | 47,630 | 2023-08-22 | 1.1.0 |
| layout-bin-packer | 5,898,960 | 46,646 | 2023-09-23 | 2.0.0 |
| ember-collection | 5,554,269 | 46,156 | 2023-11-01 | 3.0.0 |
| ember-on-resize-modifier | 12,984,327 | 45,290 | 2023-03-24 | 2.0.2 |
| @storybook/ember-cli-storybook | 522,400 | 25,934 | 2023-08-16 | 0.6.1 |
| ember-cli-bundle-analyzer | 366,499 | 24,734 | 2023-02-08 | 1.0.0 |
| ember-data-copyable | 452,650 | 13,989 | 2023-12-22 | 1.3.1 |
| @mainmatter/ember-api-actions | 499,536 | 5,921 | 2023-03-08 | 0.6.0 |

**2024**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-cli-code-coverage | 45,195,219 | 277,477 | 2024-09-24 | 3.1.0 |
| ember-classic-import-meta-glob | 3,160,080 | 66,267 | 2024-11-20 | 0.1.1 |
| ember-href-to | 9,056,871 | 51,065 | 2024-09-13 | 5.0.6 |
| ember-local-storage | 8,013,258 | 44,562 | 2024-02-06 | 2.0.7 |
| ember-cli-addon-docs-yuidoc | 7,600,068 | 37,136 | 2024-02-22 | 1.1.0 |
| @ember-intl/cp-validations | 1,108,025 | 24,246 | 2024-07-10 | 7.0.2 |
| ember-array-contains-helper | 353,464 | 17,303 | 2024-01-03 | 3.0.0 |
| ember-user-activity | 1,049,065 | 16,064 | 2024-12-04 | 8.0.1 |
| ember-hbs-minifier | 675,085 | 14,327 | 2024-11-15 | 1.3.0 |

**2025**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-test-selectors | 68,929,200 | 507,151 | 2025-04-09 | 7.1.0 |
| ember-try | 59,304,951 | 319,640 | 2025-03-07 | 4.0.0 |
| ember-wormhole | 43,204,311 | 224,588 | 2025-08-07 | 0.6.1 |
| ember-svg-jar | 31,993,848 | 152,971 | 2025-11-07 | 2.7.1 |
| ember-cp-validations | 20,507,670 | 147,281 | 2025-04-17 | 7.0.0 |
| ember-cli-showdown | 14,535,603 | 135,770 | 2025-09-26 | 9.0.2 |
| ember-tooltips | 20,393,172 | 123,008 | 2025-06-03 | 4.0.0 |
| ember-cli-clipboard | 19,339,614 | 122,288 | 2025-03-15 | 1.3.0 |
| ember-modal-dialog | 16,478,775 | 96,983 | 2025-12-08 | 5.0.0 |
| torii | 8,649,252 | 94,687 | 2025-08-07 | 1.0.0 |
| ember-ref-bucket | 12,355,749 | 80,326 | 2025-03-06 | 5.0.8 |
| ember-bootstrap | 11,219,427 | 63,722 | 2025-11-26 | 6.7.0 |
| ember-feature-flags | 6,805,755 | 49,117 | 2025-02-22 | 8.0.0 |
| ember-classy-page-object | 1,080,439 | 44,255 | 2025-01-06 | 0.8.0 |
| ember-metrics | 12,448,917 | 42,256 | 2025-02-27 | 2.0.0 |
| ember-collapsible-panel | 5,801,346 | 41,731 | 2025-07-03 | 6.1.1 |
| @percy/ember | 10,681,317 | 40,946 | 2025-07-28 | 5.0.0 |
| ember-drag-sort | 904,077 | 34,968 | 2025-10-23 | 4.2.0 |
| ember-qunit-nice-errors | 418,503 | 34,349 | 2025-08-01 | 2.0.0 |
| ember-popper-modifier | 496,550 | 22,021 | 2025-03-19 | 4.1.1 |
| ember-froala-editor | 593,310 | 15,162 | 2025-11-19 | 4.7.1 |
| ember-apollo-client | 358,165 | 13,598 | 2025-08-20 | 5.0.0 |
| ember-cli-fastboot-testing | 451,245 | 10,330 | 2025-12-03 | 0.7.0 |

**2026**

| Name | Downloads/2y | Downloads/month | Last Published | Latest Version |
|------|-----------|-----------------|----------------|----------------|
| ember-exam | 57,838,581 | 455,320 | 2026-02-17 | 10.1.0 |
| ember-data-model-fragments | 12,204,477 | 109,932 | 2026-02-27 | 7.0.3 |
| ember-engines | 11,699,199 | 59,848 | 2026-02-25 | 0.13.1 |
| ember-freestyle | 693,137 | 25,546 | 2026-03-22 | 0.23.0 |
| ember-bootstrap-power-select | 1,390,068 | 6,292 | 2026-03-06 | 6.1.0 |

</details>

<details>
<summary>

#### Filtered Out (Outdated Dependencies)

</summary>

Last published breakdown: 
2016: 4, 
2017: 7, 
2018: 14, 
2019: 16, 
2020: 11, 
2021: 14, 
2022: 11, 
2023: 7, 
2024: 7, 
2025: 6, 
2026: 7, 

**2016**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-cli-release | 2016-06-18 |  |
| ember-cli-import-polyfill | 2016-07-28 |  |
| ember-cli-sri | 2016-08-10 |  |
| ember-cli-inline-content | 2016-10-27 |  |

**2017**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-cli-node-assets | 2017-03-21 |  |
| ember-disable-prototype-extensions | 2017-07-21 |  |
| ember-cli-build-config-editor | 2017-08-27 |  |
| ember-factory-for-polyfill | 2017-11-14 |  |
| ember-runtime-enumerable-includes-polyfill | 2017-11-16 |  |
| ember-cli-legacy-blueprints | 2017-11-20 |  |
| ember-qunit-source-map | 2017-12-09 |  |

**2018**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-hash-helper-polyfill | 2018-01-04 |  |
| ember-native-dom-event-dispatcher | 2018-01-12 |  |
| ember-debug-handlers-polyfill | 2018-01-21 |  |
| ember-one-way-controls | 2018-02-26 |  |
| ember-cli-sauce | 2018-03-02 |  |
| loader.js | 2018-04-11 |  |
| ember-font-awesome | 2018-05-22 |  |
| ember-string-ishtmlsafe-polyfill | 2018-05-28 |  |
| ember-invoke-action | 2018-05-30 |  |
| ember-named-arguments-polyfill | 2018-06-06 |  |
| ember-legacy-class-shim | 2018-06-11 |  |
| @ember-decorators/argument | 2018-10-04 |  |
| ember-disable-proxy-controllers | 2018-10-18 |  |
| ember-component-inbound-actions | 2018-11-19 |  |

**2019**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-d3 | 2019-01-18 |  |
| ember-weakmap | 2019-02-21 |  |
| ember-cli-cjs-transform | 2019-02-21 |  |
| ember-on-helper | 2019-05-12 |  |
| ember-code-snippet | 2019-06-24 |  |
| ember-lodash | 2019-07-10 |  |
| ember-macro-helpers | 2019-07-22 |  |
| ember-cli-typescript-blueprints | 2019-08-30 |  |
| @ember-decorators/utils | 2019-09-06 |  |
| @ember-decorators/component | 2019-09-06 |  |
| @ember-decorators/object | 2019-09-06 |  |
| ember-decorators | 2019-09-06 |  |
| ember-modifier-manager-polyfill | 2019-09-20 |  |
| ember-class-based-modifier | 2019-09-22 |  |
| ember-uuid | 2019-10-24 |  |
| ember-export-application-global | 2019-11-14 |  |

**2020**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-route-action-helper | 2020-01-19 |  |
| ember-sinon | 2020-02-25 |  |
| ember-decorators-polyfill | 2020-03-26 |  |
| ember-cli-moment-shim | 2020-05-11 |  |
| ember-getowner-polyfill | 2020-05-18 |  |
| ember-test-waiters | 2020-07-28 |  |
| ember-popper | 2020-08-10 |  |
| ember-cache-primitive-polyfill | 2020-09-18 |  |
| ember-cli-pretender | 2020-10-09 |  |
| ember-cli-element-closest-polyfill | 2020-10-16 |  |
| ember-concurrency-decorators | 2020-12-16 |  |

**2021**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-destroyable-polyfill | 2021-01-22 |  |
| ember-native-dom-helpers | 2021-01-23 |  |
| ember-in-element-polyfill | 2021-02-05 |  |
| ember-cli-terser | 2021-04-28 |  |
| ember-concurrency-ts | 2021-05-26 |  |
| ember-cli-head | 2021-05-26 |  |
| @ember/jquery | 2021-06-10 |  |
| ember-tracked-storage-polyfill | 2021-06-14 |  |
| ember-cli-inject-live-reload | 2021-06-14 |  |
| ember-cli-dependency-lint | 2021-08-30 |  |
| ember-maybe-import-regenerator | 2021-09-07 |  |
| tracked-maps-and-sets | 2021-10-25 |  |
| ember-cli-autoprefixer | 2021-11-09 |  |
| ember-composable-helpers | 2021-12-07 |  |

**2022**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-named-blocks-polyfill | 2022-01-02 |  |
| ember-cli-content-security-policy | 2022-01-02 |  |
| ember-responsive | 2022-04-04 |  |
| @glimmer/tracking | 2022-04-11 |  |
| ember-cli-sass | 2022-04-26 |  |
| ember-asset-loader | 2022-05-09 |  |
| ember-require-module | 2022-06-27 |  |
| ember-get-config | 2022-07-01 |  |
| ember-fetch | 2022-08-23 |  |
| ember-cli-postcss | 2022-09-02 |  |
| ember-assign-polyfill | 2022-11-30 |  |

**2023**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-maybe-in-element | 2023-02-11 |  |
| ember-cached-decorator-polyfill | 2023-07-26 |  |
| ember-cli-test-loader | 2023-07-26 |  |
| ember-cli-clean-css | 2023-09-05 |  |
| @ember/legacy-built-in-components | 2023-10-11 |  |
| ember-this-fallback | 2023-10-17 |  |
| ember-compatibility-helpers | 2023-10-31 |  |

**2024**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-cli-dotenv | 2024-01-04 |  |
| ember-cli-typescript | 2024-03-05 |  |
| ember-cli-fastboot | 2024-05-21 |  |
| ember-cli-app-version | 2024-06-10 |  |
| prember | 2024-07-05 |  |
| ember-cli-mirage | 2024-09-04 |  |
| ember-cli-dependency-checker | 2024-11-13 |  |

**2025**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| @ember/render-modifiers | 2025-03-01 |  |
| ember-classic-decorator | 2025-03-21 |  |
| ember-functions-as-helper-polyfill | 2025-05-06 |  |
| ember-tether | 2025-11-30 |  |
| @embroider/util | 2025-12-02 |  |
| ember-template-imports | 2025-12-24 |  |

**2026**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-cli-babel | 2026-02-02 |  |
| @ember/optional-features | 2026-02-08 |  |
| ember-css-modules | 2026-02-25 |  |
| @embroider/macros | 2026-03-24 |  |
| ember-cli-htmlbars | 2026-03-29 |  |
| ember-auto-import | 2026-03-29 |  |
| @sentry/ember | 2026-03-31 |  |

**Other**

| Outdated Dep | Last Published | Why |
|----------------|----------------|-----|
| ember-cli-deploy* | — |  |
| *broccoli-** | — |  |

</details>

### Streamlining migration for large, complex v1/v2 addons

Migration can either be done via a codemod (Mainmatter has expressed interest in developing one) or via a manual process:

1. **Move to pnpm.** This simplifies workspace and dependency management for the remaining steps.
2. **Convert to a single-package monorepo.** Set up pnpm workspaces so the addon is a workspace package.
3. **Extract the docs to a "docs app."** Move any documentation or dummy app content into a separate `docs-app` package in the monorepo, and add its tests to CI.
4. **Extract the tests to a "test app."** Move the addon's tests into a separate `test-app` package in the monorepo, and add it to CI as well.
5. **Move `addon/` and `test-support/` to a temporary location.** Move these folders to a sub-folder or other temporary location so you can generate a fresh addon in place.
6. **Generate a new addon at the desired location.** Use `ember init --blueprint @ember/addon-blueprint` (to generate in the current directory) or `ember addon addon-name --blueprint @ember/addon-blueprint` (to start fresher in a new directory -- sometimes easier). Then copy the files from `addon/` and `test-support/` from their temporary homes into the new addon's `src/` directory. This involves changing all internal imports to relative paths, and they must use file extensions. Generating from `@ember/addon-blueprint` also gives maintainers the opportunity to re-test private functions and internal behaviors -- especially useful if any of those tests had to be disabled during step 4.
7. **Publish a major version.** This is a breaking change because consumers now need `ember-auto-import` v2, Embroider, or Vite to use the addon.
