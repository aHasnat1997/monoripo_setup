# Industry-Standard Bun Monorepo — Build Guide

A monorepo you can drop any project into: NestJS APIs, React/Next/Astro frontends,
React Native (Expo) apps — all sharing one Prisma client, one env system, one lint/tsconfig
setup. Built on **Bun workspaces + Turborepo**.

---

## 1. Why this shape

Two tools, two jobs:

- **Bun workspaces** — the package manager layer. Links your `packages/*` into every
  `apps/*` via symlinks (`node_modules/@repo/db` → `packages/db`), so a change in a
  shared package is instantly visible everywhere, no publish/version step.
- **Turborepo** — the task runner. Knows the dependency graph between packages/apps,
  runs tasks in the right order, and caches results so `bun run build` doesn't
  rebuild things that haven't changed.

This is the same pattern Vercel, Shopify, and most serious TS shops use — just with
Bun instead of npm/pnpm as the package manager.

---

## 2. Final folder structure

```
my-monorepo/
├── apps/
│   ├── api/                    # NestJS
│   ├── web/                    # Next.js
│   ├── marketing/               # Astro
│   └── mobile/                  # React Native (Expo)
├── packages/
│   ├── db/                      # Prisma schema + generated client
│   ├── ui/                      # shared React components (web + mobile-agnostic pieces)
│   ├── config/                  # shared runtime config / env schema (zod)
│   ├── types/                   # shared TS types/DTOs across api ↔ web ↔ mobile
│   ├── eslint-config/
│   └── typescript-config/       # shared tsconfig bases
├── .env                         # root, shared/default values (git-ignored)
├── .env.example
├── package.json                 # root — workspaces + shared devDeps only
├── bun.lock
├── turbo.json
├── tsconfig.base.json
└── biome.json (or .eslintrc)    # optional, see §7
```

Rule of thumb: **`apps/` = deployable things. `packages/` = shared code that isn't
deployed on its own.**

---

## 3. Bootstrapping from scratch

```bash
mkdir my-monorepo && cd my-monorepo
bun init -y
mkdir -p apps packages
```

Root `package.json` — this is the important part, it declares the workspaces and
holds only tooling that's genuinely shared:

```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "db:generate": "turbo run db:generate",
    "db:migrate": "bun run --cwd packages/db migrate"
  },
  "devDependencies": {
    "turbo": "^2.3.0",
    "typescript": "^5.7.0"
  },
  "packageManager": "bun@1.2.0"
}
```

`"private": true` at the root is mandatory — it stops anyone from accidentally
`bun publish`-ing the whole monorepo as a package.

Install Turborepo and wire the pipeline:

```bash
bun add -D turbo -E
```

`turbo.json`:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "db:generate": {
      "cache": false
    }
  }
}
```

`dependsOn: ["^build"]` means: before building an app, build every package it
depends on first. This is what makes `packages/db`'s generated Prisma client be
ready before `apps/api` tries to import it.

---

## 4. Adding apps with real framework CLIs

You don't hand-roll boilerplate — you still use each framework's official CLI, you
just point its output into `apps/`. Bun supports `bunx` the same way `npx` works.

**NestJS:**

```bash
cd apps
bunx @nestjs/cli new api --package-manager bun
cd ..
```

**Next.js:**

```bash
cd apps
bunx create-next-app@latest web --typescript --tailwind --app --use-bun
cd ..
```

**Astro:**

```bash
cd apps
bunx create-astro@latest marketing --template minimal
cd ..
```

**React Native (Expo):**

```bash
cd apps
bunx create-expo-app@latest mobile
cd ..
```

After each one, do two cleanup steps:

1. Delete the CLI-generated `node_modules` and lockfile inside that app folder
   (`rm -rf apps/api/node_modules apps/api/bun.lock` etc.) — you want **one** lockfile
   at the root, not one per app.
2. In each app's `package.json`, rename `"name"` to a scoped name, e.g.
   `"@repo/api"`, `"@repo/web"`, `"@repo/mobile"`. This is what lets other
   packages `import` from it and lets Turborepo track it as a graph node.

Then from the root:

```bash
bun install
```

This single install resolves and links **everything** — all apps, all packages —
into one root `node_modules`, with workspace packages symlinked in.

---

## 5. Installing dependencies into a _specific_ app

This is the part people get wrong — don't `cd` into the app and run `bun add`.
Use `--cwd` (or `--filter`) from the root so the lockfile stays unified:

```bash
# add express types to just the api app
bun add axios --cwd apps/api

# add a dev dependency to just web
bun add -D @types/react --cwd apps/web

# add a shared package (from packages/) as a dependency of an app
bun add @repo/db --cwd apps/api
```

`bun add @repo/db --cwd apps/api` works because Bun resolves workspace-scoped
packages automatically — no need to publish to npm, it just symlinks
`apps/api/node_modules/@repo/db → packages/db`.

To run a script that lives inside one specific app/package:

```bash
bun run --cwd apps/api start:dev
bun run --cwd packages/db migrate
```

Or via Turborepo filters (better once you have many apps, since it respects the
dependency graph and caching):

```bash
turbo run dev --filter=@repo/api
turbo run build --filter=@repo/web...   # the ... includes its dependencies too
```

---

## 6. Shared packages — the actual "hold key things" layer

### `packages/db` — Prisma, shared across every app

```
packages/db/
├── prisma/
│   └── schema.prisma
├── src/
│   └── index.ts        # exports the singleton PrismaClient
├── package.json
└── tsconfig.json
```

```bash
mkdir -p packages/db/src
bun add -D prisma --cwd packages/db
bun add @prisma/client --cwd packages/db
bun add pg @prisma/adapter-pg --cwd packages/db
bun add -D @types/pg --cwd packages/db
cd packages/db && bunx prisma init && cd ../..
```

`packages/db/package.json`:

```json
{
  "name": "@repo/db",
  "private": true,
  "version": "0.0.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "scripts": {
    "db:generate": "prisma generate",
    "migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@prisma/client": "^6.0.0"
  },
  "devDependencies": {
    "prisma": "^6.0.0"
  }
}
```

`packages/db/src/index.ts` — the important bit: a **singleton** so every app that
imports it shares one connection pool instance in dev (prevents the classic
"too many Prisma Client instances" hot-reload problem):

```ts
import { PrismaClient } from "../generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
import { Pool } from "pg";

const globalForPrisma = globalThis as typeof globalThis & {
  prisma?: PrismaClient;
  pool?: Pool;
};

const pool =
  globalForPrisma.pool ??
  new Pool({
    connectionString: process.env.DATABASE_URL,
  });

const adapter = new PrismaPg(pool);

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    adapter,
    log: process.env.NODE_ENV === "development" ? ["query"] : [],
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
  globalForPrisma.pool = pool;
}
```

Now in `apps/api`:

```bash
bun add @repo/db --cwd apps/api
```

```ts
import { prisma } from "@repo/db";
```

Same import works from any app — `apps/web`'s server actions, a NestJS service,
a background worker package, etc. One schema, one client, one source of truth.

Given you already run **multi-schema PostgreSQL** setups (as in the marketplace
and platform work), this same package is where the `multiSchema` preview feature
and all your `schema.prisma` model blocks live — every app just imports the
generated client, nobody else touches the schema file directly.

### `packages/types` — shared DTOs/interfaces

```json
{ "name": "@repo/types", "main": "src/index.ts", "types": "src/index.ts" }
```

Put request/response shapes here once, import in both `apps/api` (NestJS DTOs
can `implement` these) and `apps/web` (fetch return types) — kills the classic
frontend/backend type drift.

### `packages/ui` — shared React components

Works for `apps/web` (Next) and `apps/marketing` (Astro, via its React
integration). React Native has different primitives (`View` vs `div`), so don't
force RN into this package — give it its own `packages/ui-native` if you want
shared mobile components, or just keep design tokens shared (`packages/config`)
and let each platform render its own JSX.

### `packages/config` — env schema + shared constants

A zod-validated schema so a missing env var fails at boot, not at runtime deep
in a request handler:

```ts
// packages/config/src/env.ts
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "production", "test"]),
  REDIS_URL: z.string().url(),
});

export const env = schema.parse(process.env);
```

### `packages/eslint-config` + `packages/typescript-config`

Base configs each app extends, so you set lint/strictness rules once:

```
packages/typescript-config/base.json
packages/typescript-config/nestjs.json   # extends base, adds decorators/emitDecoratorMetadata
packages/typescript-config/nextjs.json   # extends base, adds jsx/dom lib
packages/typescript-config/react-native.json
```

Each app's `tsconfig.json`:

```json
{ "extends": "@repo/typescript-config/nextjs.json" }
```

---

## 7. Env management from the root (not per-app)

Keep **one** root `.env` (git-ignored) and one `.env.example` (committed). Every
app reads from the same file via Turborepo's env pass-through — you don't
duplicate `DATABASE_URL` into four different `apps/*/.env` files that drift out
of sync.

Two practical approaches, pick one:

**A. `dotenv-cli` at the root, injected per task (simplest, works with Bun natively):**

```bash
bun add -D dotenv-cli -w
```

```json
// root package.json scripts
"dev:api": "dotenv -e .env -- turbo run dev --filter=@repo/api"
```

**B. Turborepo's built-in env handling** — declare which env vars each task
actually depends on in `turbo.json` so caching stays correct when they change:

```json
{
  "tasks": {
    "build": {
      "env": ["DATABASE_URL", "NODE_ENV"],
      "dependsOn": ["^build"]
    }
  }
}
```

Bun itself also auto-loads `.env` from the **current working directory** — so
if you run `bun run --cwd apps/api start:dev`, Bun looks for `apps/api/.env`
first. To force root-level loading, set `NODE_OPTIONS` isn't needed — instead
just symlink or reference root env explicitly:

```bash
# apps/api/package.json
"start:dev": "bun --env-file=../../.env run src/main.ts"
```

That one flag (`--env-file=../../.env`) is the cleanest fix — every app's dev
script points at the root file explicitly, so there's exactly one file to edit
when a secret rotates.

For **production** on your Dokploy/VPS setup: don't ship `.env` files at all —
set the vars in Dokploy's environment panel per app (it already does this per
your Hoppscotch deployment), and each app's Dockerfile just expects
`process.env.X` to be present at runtime. Root `.env` stays a **local dev
only** convenience.

---

## 8. Running things day to day

```bash
bun install                              # once, from root — links everything
bun run db:generate                      # regenerate Prisma client after schema changes
turbo run dev --filter=@repo/api         # just the API
turbo run dev --filter=@repo/web         # just web
turbo run dev                            # everything with a `dev` script, in parallel
turbo run build --filter=@repo/web...    # build web + everything it depends on
```

Add convenience scripts to root `package.json`:

```json
"dev:api": "turbo run dev --filter=@repo/api",
"dev:web": "turbo run dev --filter=@repo/web",
"dev:mobile": "turbo run dev --filter=@repo/mobile"
```

---

## 9. Deploying each app independently (Dokploy)

Each `apps/*` gets its own `Dockerfile` (multi-stage, since you need root
`node_modules` context but a slim final image):

```dockerfile
# apps/api/Dockerfile
FROM oven/bun:1 AS base
WORKDIR /repo

FROM base AS deps
COPY package.json bun.lock turbo.json ./
COPY apps/api/package.json ./apps/api/package.json
COPY packages ./packages
RUN bun install --frozen-lockfile

FROM deps AS build
COPY . .
RUN bun run --cwd packages/db db:generate
RUN bunx turbo run build --filter=@repo/api

FROM oven/bun:1-slim AS runner
WORKDIR /repo
COPY --from=build /repo/apps/api/dist ./dist
COPY --from=build /repo/node_modules ./node_modules
CMD ["bun", "dist/main.js"]
```

Same pattern for `web`/`marketing`, swapping the build/start commands. Point
Dokploy at the repo root with a build context that includes the whole monorepo
(not just the `apps/api` subfolder) — the Dockerfile above assumes it's built
from repo root so it can see `packages/`.

---

## 10. Quick checklist for adding a new app later

1. `bunx <framework-cli> apps/<name>`
2. Delete its local `node_modules`/lockfile
3. Rename its `package.json` `"name"` → `@repo/<name>`
4. `bun install` from root
5. `bun add @repo/db @repo/types @repo/config --cwd apps/<name>` for whatever
   shared packages it needs
6. Point its `tsconfig.json` at `@repo/typescript-config/<preset>.json`
7. Add a `dev`/`build` script Turborepo can pick up (matching the task names in
   `turbo.json`)
8. Add a `Dockerfile` following §9 if it's going to Dokploy

That's the whole lifecycle — every new project type folds into the same
skeleton without touching the ones already there.
