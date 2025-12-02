# Karaoke Monorepo Scaffold (Turbo) — Full Code Scaffold

**What this document contains**

* Complete monorepo file tree for a production-ready Karaoke app (Next.js 16 frontend, tRPC backend, Neon/Postgres via Prisma, Sanity optional)
* Key files fully written (root package.json, turbo.json, pnpm workspace, apps/web starter, apps/api starter, packages/trpc, packages/db (Prisma), packages/types, packages/ui, packages/utils, infra scaffolding)
* Setup & run instructions

> This scaffold is aimed at senior engineers: monorepo with Turborepo, pnpm workspaces, shared types for tRPC, Prisma client, and clean separation between FE, BE, and packages.

---

## Repo file tree

```
karaoke-monorepo/
├── apps/
│   ├── web/                 # Next.js 16 (App router)
│   └── api/                 # tRPC server (Fastify) or Next.js app route alternative
├── packages/
│   ├── trpc/                # shared tRPC routers & client helper
│   ├── db/                  # prisma schema & client
│   ├── types/               # shared TS types
│   ├── ui/                  # shared UI primitives
│   └── utils/               # shared helpers (lrc parser, pitch detection)
├── infra/
│   ├── docker/
│   └── scripts/
├── .gitignore
├── pnpm-workspace.yaml
├── package.json
├── turbo.json
└── README.md
```

---

## 1) Root files

### `package.json`

```json
{
  "name": "karaoke-monorepo",
  "private": true,
  "version": "1.0.0",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "test": "turbo test",
    "prepare": "husky install"
  },
  "devDependencies": {
    "turbo": "^1.8.3",
    "pnpm": "^8.0.0",
    "prettier": "^2.8.0",
    "eslint": "^8.0.0"
  }
}
```

### `pnpm-workspace.yaml`

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'infra/*'
```

### `turbo.json`

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false
    },
    "lint": {},
    "test": {}
  }
}
```

### `.gitignore`

```
node_modules
.pnpm-store
.env
.env.*
/dist
/.turbo
**/.DS_Store
```

---

## 2) apps/web (Next.js 16 starter)

`apps/web/package.json`

```json
{
  "name": "apps-web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "^16.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@trpc/client": "^10.0.0",
    "@tanstack/react-query": "^5.0.0",
    "zustand": "^4.0.0",
    "tailwindcss": "^4.0.0"
  }
}
```

`apps/web/next.config.js`

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: { appDir: true }
}
module.exports = nextConfig
```

`apps/web/app/layout.tsx` (minimal)

```tsx
import './globals.css'
import { ReactNode } from 'react'

export const metadata = { title: 'Karaoke Cambodia' }

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

`apps/web/app/page.tsx`

```tsx
export default function Page() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Karaoke Cambodia — Web</h1>
      <p>Welcome to the frontend scaffold.</p>
    </main>
  )
}
```

`apps/web/lib/trpc.ts` (client setup)

```ts
import { createTRPCProxyClient, httpLink } from '@trpc/client'
import type { AppRouter } from 'packages/trpc'

export const trpc = createTRPCProxyClient<AppRouter>({
  links: [httpLink({ url: process.env.NEXT_PUBLIC_API_URL + '/trpc' })]
})
```

`apps/web/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

> Note: Add `tailwind.config.js` at repo root or in `apps/web` as preferred.

---

## 3) apps/api (tRPC + Fastify server)

`apps/api/package.json`

```json
{
  "name": "apps-api",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "start": "node dist/index.js",
    "build": "tsc -p tsconfig.build.json"
  },
  "dependencies": {
    "@trpc/server": "^10.0.0",
    "fastify": "^4.0.0",
    "@fastify/cors": "^8.0.0",
    "zod": "^3.20.0",
    "prisma": "^5.0.0",
    "@prisma/client": "^5.0.0"
  },
  "devDependencies": {
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

`apps/api/src/index.ts`

```ts
import Fastify from 'fastify'
import cors from '@fastify/cors'
import { appRouter } from 'packages/trpc/routers'
import { createHTTPHandler } from '@trpc/server/adapters/standalone'

const server = Fastify()

async function main() {
  await server.register(cors, { origin: true })

  const handler = createHTTPHandler({
    router: appRouter,
    createContext: () => ({})
  })

  server.all('/trpc/*', async (req, reply) => {
    // @ts-ignore
    const raw = req.raw
    await handler(raw, reply.raw)
    return reply
  })

  await server.listen({ port: 8080 })
  console.log('tRPC server listening on http://localhost:8080')
}

main().catch((err) => {
  console.error(err)
  process.exit(1)
})
```

> Note: using `@trpc/server/adapters/standalone` for Fastify. For production, consider using `fastify-http-proxy` or native integration.

---

## 4) packages/trpc (shared router types)

`packages/trpc/package.json`

```json
{
  "name": "packages-trpc",
  "version": "1.0.0",
  "main": "./index.ts",
  "types": "./index.ts"
}
```

`packages/trpc/routers/song.ts`

```ts
import { z } from 'zod'
import { router, publicProcedure } from './trpc-helpers'

export const songRouter = router({
  list: publicProcedure
    .input(z.object({ q: z.string().optional(), skip: z.number().optional(), take: z.number().optional() }))
    .query(async ({ input, ctx }) => {
      const { skip = 0, take = 20, q } = input
      // call db client from packages/db
      // simplified example returning stub
      return { items: [], total: 0 }
    }),
})
```

`packages/trpc/routers/index.ts`

```ts
import { router } from './trpc-helpers'
import { songRouter } from './song'

export const appRouter = router({
  song: songRouter,
})

export type AppRouter = typeof appRouter
```

`packages/trpc/trpc-helpers.ts`

```ts
import { initTRPC } from '@trpc/server'

const t = initTRPC.context<{}>().create()

export const router = t.router
export const publicProcedure = t.procedure
```

---

## 5) packages/db (Prisma)

`packages/db/package.json`

```json
{
  "name": "packages-db",
  "version": "1.0.0",
  "main": "./client.ts"
}
```

`packages/db/prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  favorites Favorite[]
  createdAt DateTime @default(now())
}

model Artist {
  id   String @id @default(uuid())
  name String
  songs Song[]
}

model Category {
  id   String @id @default(uuid())
  name String
  songs Song[]
}

model Song {
  id          String   @id @default(uuid())
  title       String
  artistId    String
  artist      Artist   @relation(fields: [artistId], references: [id])
  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])
  audioUrl    String
  lyricsJson  Json
  duration    Int
  isPublished Boolean  @default(true)
  createdAt   DateTime @default(now())
}

model Favorite {
  id      String @id @default(uuid())
  userId  String
  user    User   @relation(fields: [userId], references: [id])
  songId  String
  song    Song   @relation(fields: [songId], references: [id])
  createdAt DateTime @default(now())
}
```

`packages/db/client.ts`

```ts
import { PrismaClient } from '@prisma/client'

export const prisma = new PrismaClient()
```

---

## 6) packages/types

`packages/types/index.ts`

```ts
export type LyricsLine = { t: number; l: string }

export interface SongDTO {
  id: string
  title: string
  artistId: string
  categoryId: string
  audioUrl: string
  lyricsJson: LyricsLine[]
  duration: number
}
```

---

## 7) packages/ui (small example)

`packages/ui/button.tsx`

```tsx
export function Button({ children, onClick }: { children: React.ReactNode; onClick?: () => void }) {
  return (
    <button className="px-4 py-2 rounded bg-blue-600 text-white" onClick={onClick}>
      {children}
    </button>
  )
}
```

`packages/ui/index.ts`

```ts
export * from './button'
```

---

## 8) packages/utils

`packages/utils/lrc-parser.ts`

```ts
export function parseLrc(lrc: string) {
  const lines = lrc.split(/\r?\n/)
  const out: { t: number; l: string }[] = []
  const timeRe = /\[(\d+):(\d+)(?:\.(\d+))?\]/
  for (const line of lines) {
    const match = line.match(timeRe)
    if (!match) continue
    const min = Number(match[1])
    const sec = Number(match[2])
    const ms = match[3] ? Number(match[3].padEnd(3, '0')) : 0
    const t = min * 60 + sec + ms / 1000
    const text = line.replace(timeRe, '').trim()
    out.push({ t, l: text })
  }
  return out
}
```

---

## 9) infra/docker (examples)

`infra/docker/api.Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
WORKDIR /app/apps/api
RUN pnpm build
CMD ["node", "dist/index.js"]
```

`infra/scripts/seed.sh`

```bash
#!/usr/bin/env bash
pnpm -w exec prisma migrate deploy --schema=packages/db/prisma/schema.prisma
pnpm -w --filter packages-db node packages/db/seed.js
```

---

## 10) Dev & Run instructions (quick)

1. Install pnpm (if not installed): `npm i -g pnpm`
2. Install dependencies: `pnpm install`
3. Setup `.env` at repo root (copy `.env.example`)
4. Run dev: `pnpm dev` (turbo will proxy to packages configured scripts)

**Suggested per-package dev scripts** — add in each package `package.json` a `dev` script (web: `next dev`, api: `ts-node-dev ...`). Then `turbo dev` will run them concurrently if you configure `turbo.json` pipelines.

---

## 11) Next steps I can do for you (pick one):

* Generate a downloadable zip of the scaffold with all files created (ready-to-run)
* Create a GitHub repo with CI templates (GitHub Actions) and seeding
* Flesh out `apps/web` with a full karaoke player UI (lyrics sync + audio player + record button)
* Implement pitch detection example (Web Audio + YIN)

Tell me which next step you want and I will generate it.

---

*End of scaffold document.*
