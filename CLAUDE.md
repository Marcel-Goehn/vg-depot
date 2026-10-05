# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

vgdepot is a multi-tenant "Shop Builder + Inventory Management" SaaS (a cloud-native JTL-Wawi alternative). Merchants (tenants) get a storefront on a platform subdomain plus a backoffice for products, stock and orders. `SPEC.md` is the source of truth for requirements and architectural decisions, so read it before making design choices.

The repo is at an early scaffold stage: all three apps are still close to their generator templates. Much of the architecture below is **planned in SPEC.md, not yet implemented**.

## Layout and commands

One Git repo, but no root `package.json` and no npm workspaces: each app is an independent project with its own dependencies and lockfile. Run commands from inside each app directory.

### `backend/`: Kotlin + Spring Boot 4, Gradle (Kotlin DSL), Java 25 toolchain
```bash
./gradlew bootRun                                    # run
./gradlew build                                      # compile + test
./gradlew test                                       # all tests
./gradlew test --tests 'com.vgdepot.backend.SomeTest'            # one class
./gradlew test --tests 'com.vgdepot.backend.SomeTest.someMethod' # one method
```
`application.properties` has no datasource configured yet, so `@SpringBootTest` tests need a reachable Postgres once one is wired in.

### `frontend/vgdepot-admin/`: Vite + React 19 SPA (merchant backoffice)
```bash
npm run dev | build | lint | preview   # build = tsc -b && vite build
```

### `frontend/vgdepot-shopfront/`: Next.js 16 App Router (tenant storefront, SSR)
```bash
npm run dev | build | start | lint
```
**Next.js 16 has breaking changes compared with older versions.** Before writing Next.js code, read the relevant guide in `frontend/vgdepot-shopfront/node_modules/next/dist/docs/` (see that app's `AGENTS.md`). `next dev` rewrites that `AGENTS.md` block, so commit it rather than reverting it.

Neither frontend has a test runner configured yet.

## Architecture (target, per SPEC.md)

- **Modular monolith backend** with these modules: catalog, inventory, orders, invoicing, shipping, payments, storefront/pages, integrations/marketplaces. Module boundaries must be enforced (Spring Modulith or ArchUnit). No microservices.
- **Multi-tenancy:** one Postgres database with one shared schema and `tenant_id` on every tenant-owned table. Isolation comes from Row-Level Security (`FORCE ROW LEVEL SECURITY`, a non-owner app role). The tenant is set **per transaction** via `set_config('app.tenant_id', ..., true)`, which stays compatible with transaction-mode pooling. Never carry the tenant in a plain `ThreadLocal`. Background jobs must carry the tenant explicitly. A tenant registry holds the subdomain and connection info so tenants can be sharded later. Tests must prove tenant A cannot read tenant B's data.
- **Persistence:** **Spring Data JPA** (Hibernate). Don't add Spring Data JDBC. Use native SQL (`@Query(nativeQuery = true)` or `JdbcTemplate`, which the JPA starter already provides) for complex queries such as stock ledger aggregations. Flyway migrations are plain SQL. The schema is owned by Flyway, not Hibernate DDL generation. `kotlin("plugin.jpa")` and the `allOpen` block in `build.gradle.kts` make entities work with Hibernate proxies.
- **Stock** is a ledger of movements (receipt, issue, reservation, correction). Levels are always derived, never stored as a mutable number.
- **Provider abstractions:** marketplaces (eBay is the first adapter; Kaufland/Amazon come later) and payments (Stripe/PayPal, provider decision deferred; don't ask about it) sit behind generic interfaces so the core modules stay unchanged. Imports, scans and webhooks must be idempotent using client-generated IDs.
- **API:** REST with an OpenAPI contract (springdoc). Authorization is enforced in the backend only.
- **Auth:** Keycloak (OIDC with PKCE) with one shared realm for all merchant staff. The tenant comes from a claim or Organization, with roles `owner` and `staff`. The backend acts as an OAuth2 Resource Server. Shop customers (optional accounts) are managed by the backend itself, scoped per tenant. Keycloak details are still open (see SPEC.md).
- **Storefront:** the tenant is resolved from the `Host` subdomain. There is **no business logic in Next.js**, which only calls the backend. The target is `output: 'standalone'` with a shared (Redis) cache handler. Pages are Puck JSON component trees rendered with the same block components as the editor. The block components live in `vgdepot-shopfront` (Puck is installed there); there is no shared package.
- **Admin:** a Vite SPA using TanStack Router + Query with file-based routing. Pages live in `src/routes/`. The Vite plugin regenerates `src/routeTree.gen.ts` on `dev`/`build`, so never edit that file by hand. The `QueryClient` is passed as router context (typed in `src/routes/__root.tsx`), so loaders can use `context.queryClient`. `autoCodeSplitting` splits each route into its own chunk.
- **Theming:** per-tenant design tokens are shadcn CSS variables, written server-side into a `<style>` tag and referenced via Tailwind `@theme`. Never persist tenant customizations as Tailwind classes. Tenants get no custom JS/CSS.

## UI rule (strict)

Tailwind CSS v4 + shadcn/ui is the only allowed UI foundation in the admin app and the storefront blocks.
- Use existing shadcn/ui components and compositions. Don't build custom or custom-styled replacements.
- If something seems impossible with shadcn/ui, **stop and ask the user** instead of building a custom solution.
- Both apps use the shadcn `base-nova` style, which is built on **`@base-ui/react` (not Radix)**. Add components with `npx shadcn add <name>` from the app directory. `cn` is re-exported from the npm `cn` package in `lib/utils.ts`. The `@/*` path alias maps to `src/` in the admin app and to the project root in the shopfront.
