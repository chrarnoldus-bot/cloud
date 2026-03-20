# Monorepo Restructure Plan

## Goal

Refactor the repo from a flat layout (where the root is both the pnpm workspace root AND the Next.js app) into a standard pnpm monorepo with clear separation: `apps/`, `services/`, and `packages/`.

## Target Structure

```
/
├── apps/
│   ├── web/                                     # Next.js app (from root src/ + configs)
│   │   ├── src/
│   │   ├── public/
│   │   ├── tests/                               # E2E / Playwright
│   │   ├── dev/                                 # Dev utilities
│   │   ├── package.json                         # Next.js deps only
│   │   ├── next.config.mjs
│   │   ├── vercel.json
│   │   ├── tsconfig.json
│   │   ├── tsconfig.scripts.json
│   │   ├── jest.config.ts
│   │   ├── playwright.config.ts
│   │   ├── postcss.config.mjs
│   │   ├── components.json
│   │   ├── knip.ts                              # Web-specific knip config
│   │   ├── .env / .env.test / .env.development.local.example
│   │   ├── instrumentation-client.ts
│   │   ├── sentry.edge.config.ts
│   │   ├── sentry.server.config.ts
│   │   ├── mdx-components.tsx
│   │   ├── eslint.fallback.config.mjs
│   │   ├── .oxlintrc.json
│   │   ├── .madgerc
│   │   ├── .dependency-cruiser.js
│   │   └── .vercelignore
│   └── storybook/                               # Storybook (from root storybook/)
│
├── services/                                    # Cloudflare Workers (cloudflare- prefix stripped)
│   ├── cloud-agent/
│   │   └── wrapper/
│   ├── cloud-agent-next/
│   │   └── wrapper/
│   ├── gastown/                                 # was cloudflare-gastown
│   │   └── container/
│   │       └── plugin/
│   ├── deploy-infra/                            # was cloudflare-deploy-infra
│   │   ├── builder/
│   │   ├── dispatcher/
│   │   └── builder-docker-container/
│   ├── kiloclaw/
│   │   ├── controller/
│   │   └── google-setup/
│   ├── app-builder/                             # was cloudflare-app-builder
│   ├── code-review-infra/                       # was cloudflare-code-review-infra
│   ├── auto-triage-infra/                       # was cloudflare-auto-triage-infra
│   ├── auto-fix-infra/                          # was cloudflare-auto-fix-infra
│   ├── ai-attribution/                          # was cloudflare-ai-attribution
│   ├── db-proxy/                                # was cloudflare-db-proxy
│   ├── webhook-agent-ingest/                    # was cloudflare-webhook-agent-ingest
│   ├── session-ingest/                          # was cloudflare-session-ingest
│   ├── o11y/                                    # was cloudflare-o11y
│   ├── git-token-service/                       # was cloudflare-git-token-service
│   ├── security-sync/                           # was cloudflare-security-sync
│   ├── security-auto-analysis/                  # was cloudflare-security-auto-analysis
│   ├── gmail-push/                              # was cloudflare-gmail-push
│   └── images-mcp/                              # was cloudflare-images-mcp
│
├── packages/                                    # Shared libraries (existing + secret-catalog)
│   ├── db/
│   ├── worker-utils/
│   ├── encryption/
│   ├── eslint-config/
│   └── kiloclaw-secret-catalog/                 # Moved from kiloclaw/packages/secret-catalog/
│
├── package.json                                 # Lean workspace root (no Next.js deps)
├── pnpm-workspace.yaml                          # Updated workspace globs
├── turbo.json                                   # NEW: build orchestration
├── knip.ts                                      # Workspace-level knip config (if needed)
├── .github/workflows/                           # Updated path references
├── docs/ / plans/ / patches/
├── .nvmrc / .npmrc / .gitignore / flake.nix / etc.
└── ... (other repo-wide configs)
```

## Implementation Steps

### Phase 1: Create directory scaffolding

1. Create `apps/web/` and `apps/storybook/` (empty)
2. Create `services/` (empty)

### Phase 2: Move the Next.js app to apps/web/

This is the highest-risk change. Everything below must happen atomically (single commit).

**Move directories:**
- `src/` → `apps/web/src/`
- `public/` → `apps/web/public/`
- `tests/` → `apps/web/tests/`
- `dev/` → `apps/web/dev/`

**Move Next.js-specific config files:**
- `next.config.mjs` → `apps/web/next.config.mjs`
- `vercel.json` → `apps/web/vercel.json`
- `postcss.config.mjs` → `apps/web/postcss.config.mjs`
- `components.json` → `apps/web/components.json`
- `instrumentation-client.ts` → `apps/web/instrumentation-client.ts`
- `sentry.edge.config.ts` → `apps/web/sentry.edge.config.ts`
- `sentry.server.config.ts` → `apps/web/sentry.server.config.ts`
- `mdx-components.tsx` → `apps/web/mdx-components.tsx`
- `eslint.fallback.config.mjs` → `apps/web/eslint.fallback.config.mjs`
- `.oxlintrc.json` → `apps/web/.oxlintrc.json`
- `.madgerc` → `apps/web/.madgerc`
- `.dependency-cruiser.js` → `apps/web/.dependency-cruiser.js`
- `.vercelignore` → `apps/web/.vercelignore`
- `.env` → `apps/web/.env`
- `.env.test` → `apps/web/.env.test`
- `.env.development.local.example` → `apps/web/.env.development.local.example`
- `skills-lock.json` → `apps/web/skills-lock.json`

**Move test configs:**
- `jest.config.ts` → `apps/web/jest.config.ts`
- `playwright.config.ts` → `apps/web/playwright.config.ts`

**Move tsconfigs:**
- `tsconfig.json` → `apps/web/tsconfig.json`
- `tsconfig.scripts.json` → `apps/web/tsconfig.scripts.json`

**Split package.json:**
- Create `apps/web/package.json` with:
  - All Next.js app `dependencies` and `devDependencies`
  - All Next.js-specific scripts (`dev`, `build`, `start`, `lint:*`, `test`, `test:e2e`, etc.)
  - `pnpm.patchedDependencies` entries that are app-specific
- Update root `package.json` to be lean workspace root:
  - Keep `prepare` (husky), `preinstall` (only-allow pnpm)
  - Keep workspace-wide scripts (e.g., `typecheck` that runs `pnpm -r typecheck`)
  - Remove all Next.js deps
  - Keep `pnpm.patchedDependencies` only if patches apply to workspace-wide deps

**Fix path references in moved configs:**

- `apps/web/tsconfig.json`:
  - `@kilocode/db` path: `./packages/db/src/index.ts` → `../../packages/db/src/index.ts`
  - `@kilocode/encryption` path: similarly prefix with `../../`
  - `include` paths stay relative (still `src/**/*`, `tests/**/*`)
  - `.next/types/**/*.ts` stays relative

- `apps/web/tsconfig.scripts.json`:
  - Same `../../` prefix for package references

- `apps/web/jest.config.ts`:
  - `<rootDir>/packages/db/src/$1` → `<rootDir>/../../packages/db/src/$1`
  - `testPathIgnorePatterns` that reference `<rootDir>/cloud-agent/` etc. → update to `<rootDir>/../../services/` paths (with stripped `cloudflare-` prefix where applicable)

- `apps/web/playwright.config.ts`:
  - `testDir` stays `./tests/e2e` (since tests/ moves with the app)
  - webServer command may need updating

- `apps/web/.madgerc`: `baseDir: "."` stays correct

- `apps/web/.dependency-cruiser.js`: `tsConfig.fileName: 'tsconfig.json'` stays correct

- `apps/web/.prettierignore` — if moved, update `src/` references (or this may stay at root)

### Phase 3: Move Storybook to apps/storybook/

- `storybook/` → `apps/storybook/`
- Update any internal path references

### Phase 4: Move workers to services/

Move each worker directory from the repo root to `services/`, stripping the `cloudflare-` prefix:

```
cloud-agent/                         → services/cloud-agent/
cloud-agent-next/                    → services/cloud-agent-next/
cloudflare-gastown/                  → services/gastown/
cloudflare-deploy-infra/             → services/deploy-infra/
cloudflare-code-review-infra/        → services/code-review-infra/
cloudflare-auto-triage-infra/        → services/auto-triage-infra/
cloudflare-auto-fix-infra/           → services/auto-fix-infra/
cloudflare-app-builder/              → services/app-builder/
cloudflare-ai-attribution/           → services/ai-attribution/
cloudflare-db-proxy/                 → services/db-proxy/
cloudflare-webhook-agent-ingest/     → services/webhook-agent-ingest/
cloudflare-session-ingest/           → services/session-ingest/
cloudflare-o11y/                     → services/o11y/
cloudflare-git-token-service/        → services/git-token-service/
cloudflare-security-sync/            → services/security-sync/
cloudflare-security-auto-analysis/   → services/security-auto-analysis/
cloudflare-gmail-push/               → services/gmail-push/
cloudflare-images-mcp/               → services/images-mcp/
kiloclaw/                            → services/kiloclaw/
```

### Phase 5: Move kiloclaw-secret-catalog to packages/

- `services/kiloclaw/packages/secret-catalog/` → `packages/kiloclaw-secret-catalog/`
- Update the `package.json` if it has any relative path references

### Phase 6: Update pnpm-workspace.yaml

Replace the explicit list with globs + specific entries:

```yaml
packages:
  - apps/*
  - services/*
  - services/cloud-agent/wrapper
  - services/cloud-agent-next/wrapper
  - services/gastown/container
  - services/deploy-infra/builder
  - services/deploy-infra/dispatcher
  - packages/*

catalog:
  # ... (unchanged)
```

### Phase 7: Update GitHub Actions workflows

**ci.yml:**
- Update `dorny/paths-filter` paths: prefix all worker paths with `services/`
- Update `working-directory` for wrapper builds: `services/cloud-agent/wrapper`, `services/cloud-agent-next/wrapper`
- Update all worker directory references with stripped `cloudflare-` prefix
- Update `.next/cache` path: `${{ github.workspace }}/apps/web/.next/cache`
- Update Vercel commands to work from `apps/web/` directory
- Note: `pnpm --filter <name>` commands use package names, NOT paths — these don't change

**deploy-production.yml:**
- Update `.next/cache` path (3 occurrences)
- Update `dorny/paths-filter` for kiloclaw: `services/kiloclaw/**`
- Update Vercel link/build/deploy to use correct working directory

**deploy-workers.yml:**
- Update `WORKERS` array: use new `services/` paths with stripped `cloudflare-` prefix
- Update `workflow_dispatch` choices: same new paths
- Update `working-directory` references
- Update `git diff` path detection

**deploy-kiloclaw.yml:**
- Update all `kiloclaw` → `services/kiloclaw`
- Update Docker context and Dockerfile paths

**bump-openclaw.yml:**
- Update `kiloclaw/Dockerfile` → `services/kiloclaw/Dockerfile`

**chromatic.yml:**
- Update storybook references: `storybook/` → `apps/storybook/`

### Phase 8: Create root package.json (lean workspace root)

The new root package.json should contain:
- `"name": "kilocode-monorepo"` (or similar)
- `"private": true`
- `"packageManager": "pnpm@10.27.0"`
- `"engines": { "node": "^22" }`
- `"scripts"`: only workspace-wide scripts:
  - `"prepare": "husky"`
  - `"preinstall": "npx only-allow pnpm"`
  - `"typecheck": "pnpm -r typecheck"` (or delegate to turbo)
  - `"build": "turbo run build"`
  - `"test": "turbo run test"`
  - `"lint": "turbo run lint"`
- Minimal `devDependencies`: `husky`, `turbo`, shared tooling only

### Phase 9: Add Turborepo

Create `turbo.json`:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**", ".vercel/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

Install `turbo` as a root devDependency. Update root scripts to use `turbo run`.

### Phase 10: Fix .prettierignore and other repo-wide configs

- Update `.prettierignore` paths (`src/tests/sample` → `apps/web/src/tests/sample`, etc.)
- Update `.gitignore` if it references `src/`-specific patterns
- Update `knip.ts` — split into root-level (workspace-wide) and `apps/web/knip.ts` (app-specific)

### Phase 11: Verification

Run and fix:
1. `pnpm install` — verify workspace resolution works
2. `pnpm typecheck` (or `turbo run typecheck`) — all packages compile
3. `pnpm --filter web build` — Next.js build succeeds
4. `pnpm --filter web test` — Jest tests pass
5. Worker builds: spot-check a few with `pnpm --filter <name> typecheck`
6. Verify `pnpm -r lint` runs correctly

### Phase 12: External configuration (manual, post-merge)

These cannot be done in code and must be done manually:

- **Vercel dashboard**: Update "Root Directory" for `kilocode-app`, `kilocode-global-app`, and `kilocode-gateway` projects to `apps/web`
- **Sentry**: If Sentry source map upload references paths, update in Sentry project settings

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Git history fragmentation from `git mv` | Use `git mv` for all moves to preserve history tracking. Git is good at detecting renames. |
| Vercel deployment breaks | Update Vercel root directory setting IMMEDIATELY after merge. Consider coordinating merge timing. |
| pnpm resolution breaks | Run `pnpm install` after every structural change to catch issues early |
| Import paths break | The `@/*` alias stays the same (relative to tsconfig). Cross-package `workspace:*` deps use package names, not paths. |
| CI workflows fail on first run | Have the workflow updates in the same commit as the directory moves |
| `.env` not found by Next.js | Next.js loads `.env` from its project root — moving it to `apps/web/.env` is correct |

## Out of Scope (for future PRs)

- Renaming worker packages to consistent `@kilocode/*` convention
- Migrating Jest → Vitest for the Next.js app
- Unifying test frameworks across workers
- Extracting more shared code from `src/lib/` into `packages/`
- Resolving the duplicate `@kilocode/cloud-agent-wrapper` package name
