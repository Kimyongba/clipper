# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Project

Clipper is a booking application for an at-home dog-grooming service in Seoul. Customers choose a package, neighborhood, and arrival time; the backend assigns an available groomer and creates a `CLP-XXXXXX` booking reference.

## Stack

- Next.js App Router conventions running through **vinext**, not the `next` CLI.
- Vite and React Server Components deployed to Cloudflare Workers.
- Cloudflare D1 through Drizzle ORM.
- Tailwind CSS v4 and shadcn/ui.
- npm only; imports use the `@/*` alias rooted at the repository root.

## Commands

```sh
npm ci                         # install the exact package-lock.json dependencies
npm run db:migrate:local       # apply pending migrations to local D1
npm run db:seed:local          # destructively reset and seed local data
npm run dev                    # local server, normally http://localhost:3000
npx tsc --noEmit               # TypeScript type-check
npm run build                  # create Worker output in dist/; does not type-check
npm run preview                # build, then run the built Worker locally
npm run start                  # run existing dist/server/wrangler.json
npm run deploy                 # deploy existing generated Worker config
npm run cf-typegen             # regenerate worker-configuration.d.ts
```

There is no test framework, linter, or formatter. Before considering a change complete, run `npx tsc --noEmit`, run `npm run build`, and exercise the affected flow in the app.

For a fresh clone, run migration before seed. The local D1 database is stored under `.wrangler/`, which is gitignored and machine-local. The seed script starts by deleting every table's rows, including bookings.

Run `npm run build` before `start` or `deploy`, because both use the generated `dist/server/wrangler.json`. After changing bindings in `wrangler.jsonc`, run `npm run cf-typegen` and rebuild. Never hand-edit `worker-configuration.d.ts`.

## Architecture and rules

- D1 bindings come from `env` in `cloudflare:workers`, not `process.env`.
- Use `getDatabase()` from `lib/clipper/db.ts`; place reusable queries in `lib/clipper/data.ts`.
- `db/schema.ts` is the Drizzle schema source. For schema changes: edit it, run `npx drizzle-kit generate`, then run `npm run db:migrate:local`. Keep the Wrangler migration workflow; do not replace it with `drizzle-kit push`.
- Wrangler records applied D1 migrations in the database's `d1_migrations` table. `drizzle/meta/_journal.json` is Drizzle generation metadata, not Wrangler's applied-migration state.
- Timestamps use integer `timestamp_ms`; booleans use integers. Serialize `Date` values to ISO strings in the data layer before crossing the server/client boundary.
- Business times are **KST (`Asia/Seoul`)**. The current `findAvailableGroomer` implementation derives weekday and time with UTC getters; treat that as a known bug and do not propagate it.
- Monetary values are Korean won even though the column is named `price_cents`. A stored value of `65000` means `₩65,000`; do not divide it by 100.
- `wrangler.jsonc` currently uses `database_id: "local"`; remote deployment requires a real D1 database ID.

## Known gaps

- `/api/booking-options` returns packages and neighborhoods but not available slots. The booking UI therefore uses hardcoded fallback slots. Real slot availability still needs a server-backed source.
- There is no authentication. Appointment lookup relies on the booking reference.
