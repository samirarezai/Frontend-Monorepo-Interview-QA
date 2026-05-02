# Frontend Monorepo

A study guide for frontend interviews focused on **JavaScript/TypeScript monorepos**: workspaces, task runners, caching, boundaries, CI, and real failure modes.

**Configs & code:** the section [Reference: layout and minimal config (code)](#reference-layout-and-minimal-config-code) has copy-paste **`package.json`**, **`turbo.json`**, **Vite / Next / Jest / Vitest**, **Pinia**, **Angular paths**, and **Module Federation** sketches. Other chapters point back to it as **Reference** so narrative and snippets stay in sync.

---

## Table of contents

**Code / copy-paste:** [Reference: layout and minimal config](#reference-layout-and-minimal-config-code)

1. [Fundamentals](#1-fundamentals)
2. [Workspaces & package managers](#2-workspaces--package-managers-mid)
3. [Task orchestration: Turborepo, Nx, Lerna](#3-task-orchestration-turborepo-nx-lerna-mid--senior)
4. [TypeScript, builds, and libraries](#4-typescript-builds-and-libraries-mid--senior)
5. [Architecture, boundaries, and scale](#5-architecture-boundaries-and-scale-senior)
6. [CI/CD, caching, and developer experience](#6-cicd-caching-and-developer-experience-mid--senior)
7. [Dependencies, peers, and debugging](#7-dependencies-peers-and-debugging-mid--senior)
8. [Versioning, publishing, and consumers](#8-versioning-publishing-and-consumers-mid--senior)
9. [Testing, linting, and quality gates](#9-testing-linting-and-quality-gates-mid--senior)
10. [Micro-frontends and multi-app repos](#10-micro-frontends-and-multi-app-repos-senior)
11. [Design & scenario questions](#11-design--scenario-questions-senior)
12. [Tradeoffs, anti-patterns, and leadership](#12-tradeoffs-anti-patterns-and-leadership-senior)
13. [React.js monorepos](#13-reactjs-monorepos)
14. [Vue.js monorepos](#14-vuejs-monorepos)
15. [Angular monorepos](#15-angular-monorepos)

**Legend:** Questions span levels; senior answers emphasize **tradeoffs**, **graphs**, **ownership**, and **operational** impact.

---

## Reference: layout and minimal config (code)

Use this as a **mental model** when interviewers ask “what does the repo look like?” or “show the config.”

### Typical folder tree

```
acme/
├── package.json                 # root scripts; workspaces (npm/Yarn) or private meta (pnpm)
├── pnpm-workspace.yaml          # pnpm only — which folders are packages
├── turbo.json                   # optional — Turborepo task graph + cache outputs
├── tsconfig.base.json           # optional — shared TS paths / references
├── apps/
│   └── web/
│       ├── package.json
│       └── vite.config.ts       # or next.config.ts, angular.json, etc.
└── packages/
    └── ui/
        ├── package.json
        └── src/index.ts
```

### Root `package.json` — npm / Yarn workspaces

```json
{
  "name": "acme",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev --parallel",
    "lint": "turbo run lint"
  }
}
```

### `pnpm-workspace.yaml` (pnpm)

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### App depends on a local package

```json
{
  "name": "@acme/web",
  "dependencies": {
    "@acme/ui": "workspace:*"
  }
}
```

`workspace:*` means: “resolve `@acme/ui` from this repo, any compatible local version.”

### `turbo.json` — `^build` = build dependency packages first

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "lint": {},
    "test": {
      "dependsOn": ["^build"]
    }
  }
}
```

### TypeScript project references (sketch)

**`tsconfig.base.json`** (paths for the whole repo):

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@acme/ui": ["packages/ui/src/index.ts"]
    }
  }
}
```

**`packages/ui/tsconfig.json`** — library in composite graph:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
```

**`apps/web/tsconfig.json`** — app references the lib:

```json
{
  "extends": "../../tsconfig.base.json",
  "references": [{ "path": "../../packages/ui" }],
  "include": ["src"]
}
```

### `package.json` `"exports"` (stable public API)

```json
{
  "name": "@acme/ui",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./styles.css": "./dist/styles.css"
  },
  "sideEffects": ["*.css"]
}
```

### Shared React library — correct peers (avoid duplicate React)

```json
{
  "name": "@acme/ui",
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "@types/react": "^18.0.0",
    "@types/react-dom": "^18.0.0"
  }
}
```

**Anti-pattern:** putting `react` under `"dependencies"` in a library you publish or link into apps — risks a **second** React copy.

### Next.js — compile workspace packages

`next.config.ts` (or `.mjs`):

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@acme/ui", "@acme/hooks"],
};

export default nextConfig;
```

### Vite — resolve workspace package to source (dev)

`apps/web/vite.config.ts`:

```ts
import path from "node:path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@acme/ui": path.resolve(__dirname, "../../packages/ui/src"),
    },
  },
});
```

### Vitest — same alias as Vite

```ts
// vitest.config.ts — often import vite config or duplicate alias block
import path from "node:path";
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@acme/ui": path.resolve(__dirname, "../../packages/ui/src"),
    },
  },
  test: { environment: "jsdom" },
});
```

### Jest — mirror TS paths

```js
// jest.config.cjs
module.exports = {
  moduleNameMapper: {
    "^@acme/ui$": "<rootDir>/../../packages/ui/src/index.ts",
    "\\.(css|less)$": "identity-obj-proxy",
  },
  transformIgnorePatterns: [
    "/node_modules/(?!@acme)/", // allow transforming linked @acme/* if needed
  ],
};
```

### Force one version of a transitive dependency (pnpm)

`.npmrc` or root `package.json`:

```json
{
  "pnpm": {
    "overrides": {
      "react": "18.2.0",
      "react-dom": "18.2.0"
    }
  }
}
```

(Yarn: `"resolutions"`; npm: `"overrides"` — same idea.)

### Pinia — one store plugin per app

`apps/web/src/main.ts`:

```ts
import { createApp } from "vue";
import { createPinia } from "pinia";
import App from "./App.vue";

const app = createApp(App);
app.use(createPinia());
app.mount("#app");
```

Shared code lives in `packages/stores`; **do not** call `createPinia()` inside that package for production.

### Angular — path mapping to a built lib (simplified)

**`tsconfig.json`** (workspace root or app):

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@org/feature": ["dist/org/feature"],
      "@org/feature/*": ["dist/org/feature/*"]
    }
  }
}
```

Exact paths depend on **Angular CLI / Nx** output layout; CI order: **`ng build feature-lib`** before **`ng build shell-app`**.

### Module Federation (Webpack) — share React as singleton

Host `webpack.config.js` (conceptual):

```js
new ModuleFederationPlugin({
  name: "shell",
  remotes: { checkout: "checkout@http://localhost:5001/remoteEntry.js" },
  shared: {
    react: { singleton: true, requiredVersion: "18.2.0" },
    "react-dom": { singleton: true, requiredVersion: "18.2.0" },
  },
});
```

Remote uses the **same** `shared` keys so the host supplies **one** React instance.

---

## 1. Fundamentals

### Q: What is a monorepo?

**A:** A **monorepo** is a single version-controlled repository that contains **multiple projects** (applications, libraries, tooling configs). For frontend teams that usually means several **deployable apps** (web, admin, docs) plus **shared packages** (UI kit, hooks, API clients, design tokens, internal ESLint presets).

It is **not** the same as a monolith: apps can still be built and deployed independently; the repo is an **organizational and tooling** choice.

---

### Q: Why do companies use a monorepo for frontend?

**A:**

- **Atomic cross-package changes:** Rename a prop in a design-system component and update all consumers in **one PR** and one CI run.
- **Shared tooling:** One ESLint/Prettier/TS config package; consistent standards.
- **Visibility:** The full dependency graph and usages are in one clone—easier refactors and safer deprecations.
- **Onboarding:** One `git clone`, one workspace install (with caveats at huge scale).

**Senior angle:** The benefit is proportional to **how often packages co-evolve**. If teams never share code, a monorepo mainly adds coordination overhead.

---

### Q: Monorepo vs polyrepo (multi-repo)—when is polyrepo better?

**A:**

| Monorepo strengths | Polyrepo strengths |
|--------------------|--------------------|
| Cross-cutting refactors in one change | Strong **team/repo ownership** boundaries |
| Single CI story (with good filtering) | **Independent release cadence** per product |
| Easier internal API evolution | Smaller clones, clearer blast radius per repo |

**Senior:** Polyrepo often pairs with **published semver packages** and **version negotiation** between teams. Monorepo delays that cost until you need **selective publishing** or **multirepo sync** for external consumers.

---

### Q: What is a “workspace” in npm/pnpm/Yarn?

**A:** **Workspaces** link local packages: a root `package.json` declares workspace globs (e.g. `packages/*`, `apps/*`). Dependencies like `"@acme/ui": "workspace:*"` resolve to **sibling folders** on disk instead of the npm registry. The package manager creates symlinks (or hardlinks with pnpm) so Node’s resolution finds local code.

**Mid-level detail:** The root often has **no** or minimal application code; it orchestrates installs and scripts.

See **Reference** for a root **`workspaces`** array, **`pnpm-workspace.yaml`**, and an app using **`"workspace:*"`**.

---

## 2. Workspaces & package managers (mid)

### Q: npm, Yarn, and pnpm workspaces—what should you know in interviews?

**A:**

- **npm workspaces:** Built-in; hoisting behavior familiar to npm users; good defaults for smaller repos.
- **Yarn (Berry/classic) workspaces:** Mature; Plug’n’Play (PnP) in Yarn 2+ avoids `node_modules` entirely—stricter resolution, different DX and tooling compatibility story.
- **pnpm workspaces:** **Content-addressable store** + symlinked `node_modules`; **stricter** dependency visibility; excellent for large monorepos and deduplication.

**Senior:** Choice affects **CI cache keys**, **Docker layer caching**, and **compatibility** with tools that assume classic `node_modules` layout.

---

### Q: What is hoisting? What goes wrong?

**A:** **Hoisting** lifts dependencies to a higher-level `node_modules` so duplicate copies are reduced.

**Problem—phantom dependencies:** Package `A` imports `lodash` without listing it in `package.json` because another package hoisted `lodash` where it happened to resolve. Locally it works; **strict installs**, different order, or **pnpm** can break it.

**Fix:** Every package should **declare direct imports** in its own `dependencies` / `devDependencies`. Use lint rules (`import-x`, `depcheck`) where helpful.

---

### Q: What is pnpm’s “strict” node_modules and why does it matter?

**A:** Packages only see **declared dependencies** (plus their own subtree), not everything hoisted to the root. That surfaces missing declarations early—**good for correctness**—but can require fixing undeclared deps when migrating from npm/Yarn classic.

**Senior:** `public-hoist-pattern` / `shamefully-hoist` are **escape hatches** for broken tooling; prefer fixing the tool or dependency over permanent hoisting unless necessary.

---

### Q: What belongs in `dependencies` vs `devDependencies` in a library package?

**A:**

- **`dependencies`:** Anything required **at runtime** in published/bundled output (e.g. `clsx`, small runtime libs). Peers (e.g. `react`) go in **`peerDependencies`** when the host must supply one instance.
- **`devDependencies`:** Types, test runners, bundler, Storybook—needed to **develop** the package, not by consumers if the published artifact is pre-built.

**Nuance:** If the package is **consumed as TypeScript source** and not pre-bundled, consumers need compatible `typescript`/`@types` story—often documented rather than all in `dependencies`.

---

## 3. Task orchestration: Turborepo, Nx, Lerna (mid & senior)

### Q: What problem does Turborepo solve?

**A:** **Task running + caching + parallelization.** You define pipelines in `turbo.json` (e.g. `build` depends on `^build`). Turbo:

1. Builds a **task graph** across workspaces.
2. Runs tasks **topologically** (dependencies first).
3. **Hashes** inputs (files, env vars, lockfile, dependency task outputs) and **skips** work when the hash hits cache—locally or **remote cache** in CI.

**Mid:** Faster repeat `turbo run build` after switching branches. **Senior:** Remote cache **must** be scoped and secured (team tokens, Vercel/GitHub integration)—wrong cache keys cause **wrong artifacts**.

---

### Q: What does Nx offer beyond workspaces?

**A:** **Project graph**, **generators**, **executors**, **`affected`** commands, **module boundary rules**, and deep integrations with many frameworks. It is more **opinionated** than “pnpm + turbo + scripts.”

**Senior:** Nx shines when you want **enforced architecture** (tags, constraint rules) and **many similar apps** generated from blueprints.

---

### Q: Turborepo vs Nx—how do you choose?

**A:**

- **Turborepo:** Lighter; wraps existing `package.json` scripts; minimal migration; great with Vite/Next/custom setups.
- **Nx:** Strong when you want **first-class graph**, **codemods/generators**, and **enterprise-scale** policy enforcement.

Many teams use **pnpm workspaces + Turborepo**; Nx can also use Turborepo-style caching in newer setups—verify current docs for your interview year.

---

### Q: What is Lerna’s role today?

**A:** Historically: **versioning** and **publish** orchestration for many packages; often paired with workspaces. Many teams moved **task running/caching** to Turbo/Nx and **versioning** to **Changesets** or **semantic-release**. In interviews, mention **legacy** migrations and **why** you moved (speed, maintenance, graph).

---

### Q: What does `^build` mean in Turborepo pipeline `dependsOn`?

**A:** **`^`** means “workspace **dependencies** of this package must run `build` first” (upstream in the **package** graph). Without `^`, `dependsOn: ["lint"]` can mean **same-package** task ordering.

**Senior:** Confusing `^` vs same-package deps causes **subtle race conditions** in CI (app builds before its lib is built).

---

## 4. TypeScript, builds, and libraries (mid & senior)

### Q: What are TypeScript project references?

**A:** In `tsconfig.json`, **`references`** point to other `tsconfig` projects. With **`"composite": true`**, TS can **incrementally** build in dependency order and emit **`.tsbuildinfo`** for caching.

**Use case:** Many internal packages with clear build order; faster `tsc --build` in CI than one giant project.

**Senior:** References must stay in sync with **runtime** package boundaries; misconfigured `paths` without matching build outputs cause “types work in IDE but fail in CI.”

**CLI (composite build):** from repo root, build referenced projects in order:

```bash
pnpm exec tsc -b packages/ui
pnpm exec tsc -b apps/web
```

(Or one solution-specific `tsconfig.build.json` that references everything you need.)

**Reference** has a minimal **`references` + `composite`** layout.

---

### Q: Ship library as **source** vs **pre-built `dist/`**?

**A:**

| Approach | Pros | Cons |
|----------|------|------|
| **Source** (app transpiles sibling package) | Fast iteration; single Babel/SWC pipeline | App bundler must handle package’s TS/JSX; harder if consumers use different tools |
| **Pre-built** | Predictable published surface; faster app compile if heavy | Need `watch` in dev; source maps; semver for API changes |

**Senior:** **Tree-shaking** and **sideEffects** in `package.json` matter for pre-built ESM; **dual CJS/ESM** still trips teams—prefer **one module format** aligned with your ecosystem (usually ESM for greenfield).

---

### Q: What is `package.json` `"exports"` field?

**A:** A **public entry map**: which subpaths are importable, **conditions** (`import`/`require`/`default`), and optional **types** resolution. It **prevents** deep imports into `src/` and gives a **stable API surface** for a package.

**Senior:** Interviewers like hearing you **blocked** `import '@acme/ui/src/internal/foo'` on purpose.

Full **`exports`** + **`sideEffects`** example → **Reference** (`package.json` subsection).

---

## 5. Architecture, boundaries, and scale (senior)

### Q: How do you enforce module boundaries between packages?

**A:**

- **ESLint:** `import/no-restricted-paths`, `eslint-plugin-boundaries`, or **Nx enforce-module-boundaries** with tags (`scope:shared`, `type:feature`).
- **Code review + CODEOWNERS** on sensitive paths (`packages/design-system`).
- **`exports`** + no deep imports.

**Senior:** Rules should match **business domains**, not only folder names—otherwise teams work around them.

**Example (ESLint flat config)** — discourage deep imports that bypass `exports`:

```js
// eslint.config.js (sketch — adjust package name)
export default [
  {
    rules: {
      "no-restricted-imports": [
        "error",
        {
          patterns: [
            {
              group: ["@acme/ui/src/*", "@acme/ui/src/**"],
              message: "Import from @acme/ui public API (package exports), not src/.",
            },
          ],
        },
      ],
    },
  },
];
```

(Nx uses **tags** + `enforce-module-boundaries` for the same idea at graph scale.)

---

### Q: How do you handle circular dependencies between packages?

**A:**

1. **Detect** with madge, dependency-cruiser, or Nx graph.
2. **Break** by extracting a smaller **third package** (types, primitives), **inverting** the dependency (interfaces owned by the lower layer), or **merging** over-split packages.

**Senior:** Cycles often signal **wrong domain cut**—fix the model, not only the import.

---

### Q: How do you scale a monorepo to many teams?

**A:**

- **Ownership:** CODEOWNERS, service catalog / README per package.
- **Trunk + affected CI:** merge often; only run expensive tests for **affected** graph.
- **RFCs** for cross-cutting API changes to shared packages.
- **Deprecation policy:** codemods, timelines, Slack/release notes.

**Senior:** Watch for **“shared everything”**—one `utils` package becomes a junk drawer; split by **cohesion** and **release blast radius**.

---

## 6. CI/CD, caching, and developer experience (mid & senior)

### Q: How do you speed up CI in a large monorepo?

**A:**

- **Affected-only** pipelines (`nx affected`, `turbo` with `--filter` / change detection).
- **Remote build cache** (correct env vars in hash inputs).
- **Parallel jobs** following the task graph; **merge queues** for high throughput.
- Cache **pnpm store** / **Yarn cache** with stable keys (`lockfile` hash).
- **Sharding** tests; separate **E2E** per app or nightly for non-critical paths.

**Senior:** Stale cache bugs are **P0**—document which env vars participate in hashes; use **verbosity** to debug cache misses.

---

### Q: What is “affected” in Nx/Turbo terms?

**A:** Using **git diff** (baseline…head) plus the **project dependency graph**, determine which projects **changed or depend on changed** projects. CI runs `test`/`build`/`lint` only for that subgraph.

**Mid:** Faster PR checks. **Senior:** Baseline choice matters (`main` vs merge-base); **lockfile-only** changes might still warrant selective installs.

**Example commands** (shape varies by CI and default branch):

```bash
# Nx — only test projects affected by the diff vs main
npx nx affected -t test --base=origin/main --head=HEAD

# Turborepo — this app and everything it depends on
pnpm exec turbo run build --filter=@acme/web...

# Turborepo — change-based runs use your version’s flags (e.g. --affected or filter-from-ref); check turbo.build docs
pnpm exec turbo run lint test
```

Use **merge-base** with stacked PRs if your team needs a stricter baseline than `main`.

---

### Q: How do you structure root scripts vs package scripts?

**A:** Root: **`dev`**, **`build`**, **`test`**, **`lint`** delegating to turbo/npm/pnpm `-r`. Packages: **local** truth for what `build` means. Avoid duplicating logic—**compose** small scripts.

---

## 7. Dependencies, peers, and debugging (mid & senior)

### Q: “Invalid hook call” / duplicate React in a monorepo—causes and fixes?

**A:** **Multiple React instances** in the bundle (different physical copies). Common causes:

- Library bundles **React** instead of marking it **peer**.
- **Nested** `node_modules` with another React.
- **Symlink** + bundler resolving two different paths to “different” modules.

**Fixes:** `react`/`react-dom` as **peers** in libraries; align versions at root; **`pnpm.overrides`** / resolutions when needed; ensure **one** resolution path in the bundler (Next/Vite config).

---

### Q: What are `overrides` / `pnpm.overrides` / Yarn `resolutions`?

**A:** Force a **single version** of a transitive dependency for the whole tree—useful for security patches or **deduping** broken duplicates. **Risk:** masks upstream bugs; document why they exist and **remove** when fixed upstream.

---

### Q: How do you debug “works on my machine” in workspaces?

**A:**

- `pnpm why <pkg>` / `npm ls` / `yarn why`.
- Clear caches; **reinstall** from lockfile.
- Compare **Node version** (`.nvmrc` / Volta / asdf).
- Compare **env vars** used in Turbo hash.
- **CI repro** with `act` or minimal docker image.

---

## 8. Versioning, publishing, and consumers (mid & senior)

### Q: How do internal-only packages version?

**A:** Often **`workspace:*`** in dependents; no publish step. Optional **0.0.0** or calendar versions for packages that **are** published later.

```json
{
  "dependencies": {
    "@acme/ui": "workspace:*",
    "@acme/utils": "workspace:^"
  }
}
```

`*` = any local version; `^` = compatible local range (pnpm supports both; exact semantics follow package manager docs).

---

### Q: What is Changesets (high level)?

**A:** Developers add markdown **changeset** files (patch/minor/major intent). Tool opens a **“Version packages”** PR bumping versions and **changelogs** consistently across interdependent packages—good for **npm** consumers.

---

### Q: When do you use semantic-release instead?

**A:** **Fully automated** semver from conventional commits; great for **libraries** with strong commit discipline. Less ideal if product teams don’t want commit-message law.

---

## 9. Testing, linting, and quality gates (mid & senior)

### Q: Where should tests live?

**A:** **Unit** tests next to code in each package; **integration** at app boundary or dedicated `e2e/`; **visual** regression (Chromatic/Percy) often tied to **design system** package.

**Senior:** Shared **`@acme/test-utils`** package—keep it **small** and **framework-agnostic** where possible to avoid circular deps.

---

### Q: How do you share ESLint/Prettier config?

**A:** **`@acme/eslint-config`** package exporting flat config or legacy extends; apps `extends` that package. **Single bump** updates all consumers—plan migration windows.

**Consumer `eslint.config.mjs`** (flat config — sketch):

```js
import acme from "@acme/eslint-config";

export default [...acme];
```

**Shared package** `@acme/eslint-config` exports an array of config objects (or a preset factory) so every app stays aligned.

---

### Q: What quality gates on PRs?

**A:** Lint, typecheck, unit tests, **affected** integration tests, bundle size budgets (optional), **lockfile** integrity. **Senior:** Balance **merge time** vs **risk**; use **path filters** for docs-only PRs.

---

## 10. Micro-frontends and multi-app repos (senior)

### Q: Monorepo vs micro-frontends?

**A:** **Monorepo** = **source/repo** layout. **Micro-frontends** = **runtime** composition (multiple independently deployed UIs in one shell). They are orthogonal: you can have **many apps in one monorepo**, each deployed separately (sometimes called **multi-repo style deploy, monorepo source**).

---

### Q: Runtime integration patterns you should name?

**A:** **Webpack Module Federation**, **single-spa**, **iframes** (isolation, worse UX), **edge includes** (ESI), **server-driven** partials. Tradeoffs: **shared vendor** vs isolation, **version skew**, **routing**, **auth** handoff.

---

## 11. Design & scenario questions (senior)

### Q: “We have 40 packages and CI takes 45 minutes—what do you do?”

**A:** Measure: **what** is slow (install, build, tests, E2E). Add **remote cache** + **affected** runs; split **heavy E2E** to nightly or labels; **parallelize**; cache **install**; consider **pre-built** libs; **shard** tests; remove redundant **full builds** on docs-only changes with path filters.

---

### Q: “Design system broke two apps—how do you prevent recurrence?”

**A:** **Stricter semver** or **canary** releases; **visual regression**; **consumer contract tests** (testing-library smoke per app); **RFC** for breaking API; **codemods**; optional **feature flags** for risky components.

---

### Q: “Team A needs React 18, Team B stuck on 17—monorepo?”

**A:** Hard in **one deployable**; easier if **two apps** with separate `node_modules` trees—still painful for **shared** UI package. Real solutions: **align** versions (best), **split** package lines (`@acme/ui-v17`), or **duplicate** with cost. **Senior:** This is a **product/platform** decision, not only tooling.

---

## 12. Tradeoffs, anti-patterns, and leadership (senior)

### Q: Common monorepo anti-patterns?

**A:**

- **`@acme/utils` god package** with unrelated helpers.
- **Skipping peerDeps** and bundling framework into libraries.
- **Deep imports** bypassing `exports`.
- **No ownership** on shared packages → **merge paralysis** or **breaking changes**.
- **CI without affected** → everyone pays for every change.

---

### Q: How do you communicate a breaking change across the monorepo?

**A:** Changelog, **Slack/announce**, **migration guide**, **codemod** or scripted fix, **deprecation window** (`console.warn` in dev), **semver** if published. For internal-only, **one PR** updating all consumers is often acceptable if CI is fast enough.

---

### Q: What would you document in a monorepo README for new hires?

**A:** Prereqs (Node, pnpm), **bootstrap** commands, **how to run one app**, **how to add a package**, **boundary rules**, **where CI is configured**, **how caching works**, **who owns** design system / platform.

---

## 13. React.js monorepos

### Q: How is a React monorepo typically structured?

**A:** Common layout: `apps/web`, `apps/admin`, `apps/docs` (Next.js, Vite + React, Remix, etc.) and `packages/ui`, `packages/hooks`, `packages/api-client`, `packages/eslint-config`. Each app has its own `package.json` with `dependencies` pointing at workspace packages via `workspace:*`. The **host app** owns the bundler (Vite, Webpack, Turbopack) that transpiles JSX/TSX—shared packages are either **consumed as source** (bundler compiles them) or as **pre-built** `dist/` with proper `exports` and `types`.

---

### Q: What `peerDependencies` should a shared React component library declare?

**A:** At minimum **`react`** and usually **`react-dom`** (and **`react/jsx-runtime`** is covered by the `react` package). Optionally align **`@types/react`** / **`@types/react-dom`** as devDependencies in the library while the app supplies the runtime versions. **Never** bundle React into the library for app consumption—it causes **duplicate React** and **“Invalid hook call”**.

See **Reference** for a full **`peerDependencies` + `devDependencies`** example. **Wrong** (often causes duplicate React):

```json
{
  "dependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

---

### Q: What causes “Invalid hook call” inside a monorepo, and how do you fix it?

**A:** Almost always **two different React instances** (different physical copies or resolved paths). In monorepos: a package mistakenly lists `react` in **`dependencies`** instead of **`peerDependencies`**, nested `node_modules`, or the bundler resolving `react` from both `apps/web/node_modules` and `packages/ui/node_modules`. **Fix:** peers only, single version at root, `pnpm.overrides` / resolutions if needed, `resolve.dedupe` in Vite, or Next **transpilePackages** so one React graph is used.

---

### Q: What is Next.js `transpilePackages` in a monorepo?

**A:** Next can **compile** listed local/workspace packages through its SWC/Babel pipeline instead of expecting only pre-compiled `node_modules`. That keeps **one** React/JSX runtime and fixes packages that ship modern syntax the app must downlevel. **Senior:** List only what needs transpiling—each entry adds build work; keep shared UI **compatible** with the app’s targets.

See **Reference** for `transpilePackages` in `next.config`. **Vite dedupe** (alternative angle in interviews):

```ts
// vite.config.ts
export default defineConfig({
  resolve: { dedupe: ["react", "react-dom"] },
});
```

---

### Q: How do Jest or Vitest resolve workspace React packages?

**A:** **Vitest** inherits Vite’s `resolve.alias`—often you map `@acme/ui` to `../packages/ui/src`. **Jest** needs **`moduleNameMapper`** mirroring TS `paths`, and for CSS/assets **`moduleNameMapper`** stubs. **transformIgnorePatterns**: if a linked package ships ESM in `node_modules`, you may need to **allow** transforming that path. **Mid:** Tests fail with “Cannot find module '@acme/…'” → path map drift between TS, Vite, and Jest.

Full **alias / `moduleNameMapper`** examples are in **Reference** above.

---

### Q: How does React Fast Refresh behave with workspace packages?

**A:** When the app **transpiles source** from `packages/ui`, edits there usually **hot-reload** like app files. **Pre-built** packages often require a **watch build** (`tsup --watch`) and a full refresh or HMR boundary depending on bundler. **Interview tip:** Mention **stable file locations** and **not** mixing CJS/ESM in ways that break HMR.

---

### Q: Storybook for a design system in a React monorepo—what breaks?

**A:** Storybook has its own **Webpack/Vite** config—it must **resolve** workspace aliases, **transpile** local packages, and use the **same** React instance as stories. **Themes/tailwind:** `content` globs must include `../../packages/ui/src/**/*.tsx`. **Senior:** Duplicate Storybook per app vs **one** root Storybook—tradeoff between isolation and maintenance.

**Tailwind v3 `content` paths** (monorepo-friendly):

```js
// tailwind.config.cjs
module.exports = {
  content: [
    "./src/**/*.{js,ts,jsx,tsx}",
    "../../packages/ui/src/**/*.{js,ts,jsx,tsx}",
  ],
};
```

---

### Q: Module Federation + React in a monorepo—what do you say in an interview?

**A:** **Host** and **remotes** can live as separate `apps/*` packages; **shared** config lists `react`/`react-dom` as **singletons** so remotes do not ship another React. **Version skew** between remotes is a **runtime** problem—semver discipline and **build-time** alignment from the same lockfile help. **Senior:** Federation is **not** “free micro-frontends”—operational complexity (routing, auth, error boundaries) still lives with the team.

Minimal **`shared`** idea (Webpack) — full sample in **Reference**:

```js
shared: {
  react: { singleton: true, requiredVersion: "18.2.0" },
  "react-dom": { singleton: true, requiredVersion: "18.2.0" },
}
```

---

### Q: Can you mix Next.js App Router (RSC) and a plain Vite SPA in one monorepo?

**A:** Yes as **sibling apps**; **shared** code must respect **server vs client** boundaries: no hooks in Server Components, **`"use client"`** boundaries when a package mixes patterns. **Senior:** Extract **server-safe** utilities to a package with **no** client-only imports; otherwise you leak client bundles into RSC or break the graph.

**`"use client"`** at the top of a file that uses hooks (or re-exports client components):

```tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN((x) => x + 1)}>{n}</button>;
}
```

Pure utilities (no React, no browser APIs) can live in a package imported from **both** RSC and client without this directive.

---

### Q: React 18 + concurrent features—monorepo implications?

**A:** **Strict Mode** double effects in dev can surface **buggy** shared hooks used by multiple apps—good for quality. **`startTransition`** / **Suspense** boundaries: design-system components should **document** async-friendly patterns. **Senior:** Align **`react`** and **`react-dom`** versions across **all** apps before upgrading shared libraries.

---

## 14. Vue.js monorepos

### Q: How is a Vue 3 monorepo usually organized?

**A:** Similar to React: `apps/*` for **Vite + Vue** SPAs or admin tools, `packages/*` for **component libraries**, composables, API clients, and shared **Pinia**-agnostic types. **Vite** is the default bundler; workspace protocol links packages. **Vue** should be a **`peerDependency`** of libraries so every app uses **one** runtime copy (same duplicate-instance class of bugs as React, though Vue’s symptoms differ).

---

### Q: What should a shared Vue component library expose in `package.json`?

**A:** **`peerDependencies`:** `vue` (and optionally `vue-router` if the lib needs it). **`devDependencies`:** `vue`, `vite`, `@vitejs/plugin-vue`, `typescript`, `vue-tsc` for local build/test. **`exports`** map `./style.css` if consumers must import compiled styles. **Senior:** If shipping **SSR** components, document **`clientOnly`** patterns and **Nuxt** compatibility.

```json
{
  "name": "@acme/vue-ui",
  "type": "module",
  "peerDependencies": { "vue": "^3.4.0" },
  "devDependencies": {
    "vue": "^3.4.0",
    "vite": "^5.0.0",
    "@vitejs/plugin-vue": "^5.0.0",
    "typescript": "^5.0.0",
    "vue-tsc": "^2.0.0"
  },
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./style.css": "./dist/style.css"
  }
}
```

---

### Q: How do you share Pinia stores across apps in a monorepo?

**A:** Put **store definitions** and **types** in `packages/stores` but **instantiate** `createPinia()` in each **app** entry—Pinia is app-scoped. **Anti-pattern:** importing a singleton store module that assumes one global app in **tests** and **SSR**—breaks isolation. **Senior:** Boundaries: **domain stores** vs **UI state**; avoid circular imports between stores and feature packages.

`createPinia()` example is in **Reference**. **Store factory** in a package (sketch):

```ts
// packages/stores/src/cart.ts
import { defineStore } from "pinia";

export const useCartStore = defineStore("cart", {
  state: () => ({ items: [] as string[] }),
});
```

---

### Q: Vite + workspace packages—`optimizeDeps` and `ssr.noExternal`?

**A:** Linked workspace packages sometimes need **`server.fs.allow`** for paths outside app root. **`optimizeDeps.include`** forces pre-bundling of a linked package when discovery fails. **`ssr.noExternal`** treats workspace packages as **source** for SSR instead of externalized CJS—critical for **Nuxt** / SSR apps consuming internal libs. **Mid:** “Module not found” in dev often points to **monorepo path** or **optimizeDeps** issues.

```ts
// apps/web/vite.config.ts
import path from "node:path";
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      "@acme/vue-ui": path.resolve(__dirname, "../../packages/vue-ui/src"),
    },
  },
  server: {
    fs: { allow: [path.resolve(__dirname, "../..")] },
  },
  optimizeDeps: {
    include: ["@acme/vue-ui"],
  },
  ssr: {
    noExternal: ["@acme/vue-ui"],
  },
});
```

---

### Q: Nuxt in a monorepo—what is different from a plain Vite Vue app?

**A:** **Nuxt layers** and **`extends`** can pull shared config from a workspace package. **Auto-imports** and **composables** paths may need **`nuxt.config`** `alias` or **`srcDir`** alignment. **Senior:** **Nitro** server deps vs client bundles—shared packages must not import **Node-only** APIs into client code. **Testing:** `@nuxt/test-utils` per app; shared utils in a **non-Nuxt** package where possible.

```ts
// apps/marketing/nuxt.config.ts — consume a local layer package
import { defineNuxtConfig } from "nuxt/config";

export default defineNuxtConfig({
  extends: ["../../packages/nuxt-config-layer"],
});
```

---

### Q: Vue 2 and Vue 3 in the same monorepo—feasible?

**A:** **Technically** yes (separate apps, separate dependency trees); **practically** expensive—no shared **SFC** component library without **bridges** (@vue/compat) or duplicated UI. **Interview answer:** Prefer **migration** or **hard split** repos; if coexisting, **strict** package naming (`@acme/legacy-ui-vue2`) and **CI filters** so Vue2 apps do not build Vue3 packages.

---

### Q: How do you test Vue SFCs from a workspace package?

**A:** **Vitest** + **`@vue/test-utils`** in the library package or consumer app; Vite config with **`@vitejs/plugin-vue`**. **Vue-tsc** for type-check in CI (`vue-tsc --noEmit`). **Senior:** **Visual regression** (Chromatic/Percy) for design systems—same idea as React.

**`packages/vue-ui/vitest.config.ts`** (minimal):

```ts
import path from "node:path";
import { defineConfig } from "vitest/config";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
  test: {
    environment: "jsdom",
    globals: true,
  },
});
```

```ts
// packages/vue-ui/src/__tests__/Hello.spec.ts
import { mount } from "@vue/test-utils";
import { describe, it, expect } from "vitest";
import Hello from "../Hello.vue";

describe("Hello", () => {
  it("renders", () => {
    expect(mount(Hello).text()).toContain("Hello");
  });
});
```

---

### Q: Vue monorepo + TypeScript path aliases—one pitfall?

**A:** **`vue-tsc`**, **Vite**, and **IDE** must agree on `paths`. Mismatches show up as **green in editor, red in CI**. Prefer **`exports`** in published-style packages over deep `src` imports.

---

## 15. Angular monorepos

### Q: Why is Angular often discussed together with Nx?

**A:** **Nx** grew from Angular tooling culture; first-class **Angular plugin**, **generators** (`library`, `application`), **module boundary rules**, and **affected** commands map well to **`angular.json`/`project.json`**-style multi-project workspaces. Many enterprise Angular shops use **Nx + Angular** in one monorepo story—be ready to name **Nx** even if the question says “Angular workspace.”

---

### Q: What is an Angular “workspace” vs a generic JS monorepo?

**A:** Angular CLI creates a **multi-project workspace**: **`angular.json`** (or **`workspace.json`** in Nx) lists **applications** and **libraries** with **builders** (`build`, `test`, `lint`). **Libraries** live under `projects/my-lib` and are built with **`ng-packagr`** into **`dist/`** for consumption. **Senior:** This is more **prescriptive** than “any Vite app”—**builders** and **TS path mappings** are the contract.

---

### Q: How are internal Angular libraries consumed by apps in the same repo?

**A:** **`tsconfig` path mappings** point `@org/feature` → `dist/org/feature` or source paths during dev (tool-dependent). **`ng build my-lib`** produces **FESM** bundles and **typings**. Apps import **`@org/feature`** in TS; the CLI resolves via paths until publish. **Mid:** After lib changes, CI must **`build` libraries** before apps that depend on them—same **topological** idea as Turbo `^build`.

See **Reference** for a minimal `paths` block. **Turbo** enforces build order:

```json
{
  "tasks": {
    "build": { "dependsOn": ["^build"] }
  }
}
```

---

### Q: What are secondary entry points in Angular packages?

**A:** Beyond the main **`public-api.ts`**, libraries can expose **`my-lib/subpath`** via **`ng-package.json`** / folder structure for **tree-shakable** imports and **clear API surface**. **Senior:** Aligns with **barrel** discipline—avoid huge barrels that harm **tree-shaking** and **build time**.

**`ng-package.json`** (simplified — secondary entry `billing`):

```json
{
  "$schema": "../../node_modules/ng-packagr/ng-package.schema.json",
  "dest": "../../dist/org/feature",
  "lib": {
    "entryFile": "src/public-api.ts"
  }
}
```

Consumers import `@org/feature` or `@org/feature/billing` depending on how secondary entry points are configured (folder + `package.json` per entry in real setups — CLI generators scaffold this).

---

### Q: Angular libraries and **peerDependencies**?

**A:** **`@angular/core`**, **`@angular/common`**, often **RxJS**, should be **peers** so the **application** provides a **single** Angular instance—**multiple `@angular/core`** versions cause obscure **DI** and **NgModule** errors. **Senior:** **`engines`** or **`peerDependenciesMeta`** may appear in larger orgs for optional peers.

```json
{
  "name": "@org/ui",
  "peerDependencies": {
    "@angular/common": "^17.0.0",
    "@angular/core": "^17.0.0",
    "rxjs": "^7.4.0"
  },
  "devDependencies": {
    "@angular/common": "^17.0.0",
    "@angular/core": "^17.0.0",
    "rxjs": "^7.8.0",
    "ng-packagr": "^17.0.0"
  }
}
```

---

### Q: Standalone components vs NgModules in a monorepo—interview angle?

**A:** **Standalone** reduces **NgModule** ceremony; shared libraries increasingly export **standalone** components and **`importProvidersFrom`** only when needed. **Monorepo benefit:** migrate **app-by-app** or **lib-by-lib**; **boundary rules** (Nx) can forbid new NgModules in greenfield paths. **Senior:** **Testing** is simpler with standalone + **`TestBed.configureTestingModule({ imports: [...] })`**.

```ts
// packages/ui/src/lib/hello.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "org-hello",
  standalone: true,
  template: `<p>Hello</p>`,
})
export class HelloComponent {}
```

```ts
// consumer app
import { HelloComponent } from "@org/ui";

@Component({
  standalone: true,
  imports: [HelloComponent],
  template: `<org-hello />`,
})
export class PageComponent {}
```

---

### Q: Zone.js, SSR (Angular Universal), and shared packages?

**A:** Shared code must avoid **browser-only** APIs in **isomorphic** packages or guard with **`isPlatformBrowser`**. **SSR** builds pull **server** config from **`angular.json`**; workspace libs used on server must be **SSR-safe**. **Senior:** **Hydration** mismatches often trace to **non-deterministic** UI in shared components—fix at design-system level.

---

### Q: How does RxJS versioning work across Angular apps in one repo?

**A:** **Single lockfile** → one RxJS version wins; libraries should use **peer** range compatible with the **Angular** version’s supported RxJS. **Senior:** Breaking RxJS major with mixed apps is painful—coordinate **Angular upgrade** waves.

---

### Q: Migrating Angular major (e.g. 15 → 17) in a monorepo—how do you describe the approach?

**A:** **`ng update`** per project or orchestrated; **Nx migration** generators; **one branch** updating **shared libs first**, then **apps**; **lint + test** gates per project. **Senior:** **Incremental** strictness (TypeScript, **ESLint** `@angular-eslint`), **remove** deprecated APIs in **libraries** before the last app migrates.

---

### Q: Can Angular and React live in one monorepo?

**A:** **Yes** for **platform / migration** scenarios (separate `apps/angular-shell` and `apps/react-widget`)—often with **Module Federation** or **iframes** for runtime integration, **not** a single blended bundle. **Senior:** **Separate** lint/tsconfig/build pipelines; **document** why (acquisition, strangler pattern).

---

### Q: Angular + Turborepo without Nx—what would you mention?

**A:** **`turbo.json`** `pipeline` with **`dependsOn`: ["^build"]`** for lib→app order; each project keeps **`ng`** scripts. You lose **Nx generators/boundaries** unless you add **ESLint** rules manually. **Tradeoff:** simpler stack vs less **governance**.

---

## Quick revision checklist

| Topic | Mid-level | Senior |
|-------|-----------|--------|
| Workspaces | Symlinks, `workspace:*`, hoisting | pnpm layout, hoist escape hatches, Docker/CI cache |
| Turbo/Nx | `dependsOn`, cache, parallel | Remote cache correctness, graph debugging |
| TS | `paths`, composite | References vs bundler reality, `exports` |
| React | Peers, `transpilePackages`, Jest/Vitest paths | RSC boundaries, MF `shared` singletons, Fast Refresh |
| Vue | Vite + `plugin-vue`, Pinia app scope | `optimizeDeps` / `ssr.noExternal`, Nuxt layers, Vue2/3 split |
| Angular | `angular.json` libs, ng-packagr, path maps | Peers (`@angular/core`), secondary entrypoints, Nx + affected |
| Process | Scripts, tests | Ownership, RFCs, affected CI, incident prevention |

---

## Optional: one-liners for “any questions for us?”

- How do you handle **affected** CI baselines with **long-lived branches**?
- Who **owns** the design system package and breaking changes?
- **Remote cache** provider and **disaster** story if cache is poisoned?

---

