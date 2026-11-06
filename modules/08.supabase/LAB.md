---
duration: 2h
objectives:
  - Run Supabase locally with the CLI and Docker
  - Model project data with SQL migrations and seeds
  - Query Postgres from the right components, secured by row-level security
  - Provision a hosted Supabase and keep it in sync with migrations
  - Back every stored feature of the project with the database
---

# Lab: Supabase as your backend

Your items are hardcoded in a module and your contact form logs to the console. Time for a real backend: a local Supabase running in Docker for development - fully offline - with the schema versioned as migrations in Git, and a hosted project on supabase.com behind your Vercel deployment.

## Prerequisites

The Docker CLI with a running engine, no Docker Desktop:

- Linux: [Docker Engine](https://docs.docker.com/engine/install/), with the [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) to run `docker` without `sudo`.
- Windows: the same Docker Engine installation, inside your WSL distribution.
- macOS: the `docker` CLI with [Colima](https://github.com/abiosoft/colima) as the engine.

Verify: `docker run --rm hello-world` succeeds.

## 1. Run Supabase locally

Follow the [CLI getting started guide](https://supabase.com/docs/guides/local-development/cli/getting-started): install the CLI, run `supabase init` at your repository root (the generated `supabase/` folder is committed), then `supabase start`.

Verify:

- `supabase status` prints the local URLs and keys
- [Supabase Studio](http://localhost:54323) opens and shows an empty database

Everything runs on your machine: no account, no network needed from here until part 5.

## 2. Model your data with migrations

No click-only schema: your tables are **code, versioned in Git**, so any teammate (or your future self) can rebuild the database from scratch.

1. Design the tables for your project data, consistent with your `docs/ARCHITECTURE.md`.
2. Write them as [migrations](https://supabase.com/docs/guides/deployment/database-migrations), applied with `supabase db reset`. Prototyping in Studio is fine as long as you capture the result as a migration with [`supabase db diff`](https://supabase.com/docs/guides/deployment/database-migrations#diffing-changes).
3. [Seed](https://supabase.com/docs/guides/local-development/seeding-your-database) dummy rows in `supabase/seed.sql` - migrate the content of your hardcoded module there.

Verify: `supabase db reset` rebuilds the whole database (schema + seed) without an error, and the rows show in Studio.

## 3. Plug Next.js on Postgres

1. Install [`@supabase/supabase-js`](https://supabase.com/docs/reference/javascript/initializing) in your application and create a single client module (eg `lib/supabase.js`). Configure `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` as [environment variables](https://nextjs.org/docs/app/guides/environment-variables) in `app/.env.local` with the values printed by `supabase status` - and never commit this file.
2. Replace your hardcoded data: the collection page and the item page now [select](https://supabase.com/docs/reference/javascript/select) from the database, [fetched in server components](https://nextjs.org/docs/app/getting-started/fetching-data#server-components). Keep the [`notFound()`](https://nextjs.org/docs/app/api-reference/functions/not-found) behavior for unknown items.
3. Wire the lab 7 contact form: on submit, [insert](https://supabase.com/docs/reference/javascript/insert) into your `contacts` table. Keep the validation and success states.
4. Revisit `docs/RENDERING.md` (lab 6): a page baked at build time does not see new rows. Adjust the strategies ([`revalidate`](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate), [`generateStaticParams`](https://nextjs.org/docs/app/api-reference/functions/generate-static-params), ...) if your decisions no longer hold, and check with `npm run build`.

> [!NOTE]
> The data queries stay in server components, the contact form stays a client component - the module 6 boundary doesn't move. Module 9 introduces [`@supabase/ssr`](https://supabase.com/docs/guides/auth/server-side/nextjs) for authenticated requests; the plain client is enough for now.

## 4. Row-level security

Read the [row-level security guide](https://supabase.com/docs/guides/database/postgres/row-level-security). Your publishable key ships to every visitor: the policies are the only thing standing between the Internet and your tables.

In a new migration:

1. Enable RLS on **every** table.
2. Write the policies implementing your rules. For the blog example: articles are readable by everyone and writable by no one; contacts can be inserted by everyone and read by no one.

Verify from the application, not from Studio (Studio queries bypass RLS):

- The pages still render and the form still inserts
- A `select` on `contacts` from the application returns no rows
- Drop one policy (`supabase db reset` after editing): the matching feature must break

## 5. Provision supabase.com

Your Vercel deployment still has no database behind it. Supabase hosts [small projects for free](https://supabase.com/pricing).

1. Create a project on [supabase.com](https://supabase.com/dashboard), then [link it and push your migrations](https://supabase.com/docs/guides/deployment/database-migrations#deploy-your-project) with the CLI. Your hosted schema is now exactly the one in Git - seed a few rows from the dashboard SQL editor if needed.
2. In the [Vercel project settings](https://vercel.com/docs/environment-variables), set `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` to the hosted project values ([API settings](https://supabase.com/dashboard/project/_/settings/api-keys)), and redeploy.
3. Verify on the deployed URL: data renders, the contact form inserts, and RLS still applies.

This is your workflow from now on: develop offline against the local stack, commit migrations, `db push` when ready.

## 6. Back your whole project

So far only the items and the contact form use the database. Go ahead: every feature of your project that needs storage - comments, likes, ratings, bookings, whatever your `docs/ARCHITECTURE.md` describes - goes through the same process:

1. A migration with the table **and** its RLS policies, plus seed rows.
2. The queries from the right component: server components for reads, client components for user input (lab 6 and lab 7 boundaries).
3. The rendering strategy updated in `docs/RENDERING.md`.
4. `supabase db push` and a check on the deployed URL.

Verify: no hardcoded data module remains, and `supabase db reset` followed by `npm run build` gives a working application.

> [!NOTE]
> Features restricted to a signed-in user (own profile, private data, ...) need authentication, coming in module 9. Model their tables now, and keep their policies closed until then.

## 7. Bonus - realtime

Make one feature live: other visitors see new rows without reloading. Subscribe to [Postgres changes](https://supabase.com/docs/guides/realtime/postgres-changes) from a client component, and enable realtime on the table in a migration rather than in the dashboard.

---

*The content of this document, including all text, images, and associated materials, is the exclusive property of Adaltas and is protected by applicable copyright laws. Unauthorized distribution, reproduction, or sharing of this content, in whole or in part, is strictly prohibited without the express written consent of the author(s). Any violation of this restriction may result in legal action and the imposition of penalties as prescribed by law.*
