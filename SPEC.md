# Shop Builder + Inventory Management SaaS

I'm building a multi-tenant "Shop Builder + Inventory Management" SaaS platform, positioned as a modern, cloud-native alternative to JTL-Wawi (German e-commerce ERP) for small and medium merchants.

In this platform, merchants (our customers, "tenants") sign up and automatically receive a standardized storefront on a platform subdomain that they can fully customize themselves, plus a backoffice to manage products, stock and orders. The merchants' own customers buy through the storefront.

## Scope of the MVP

Authenticated merchant users (with roles: `owner` and `staff`) will be able to:

- Create, view, update and delete products (incl. variants, images, prices, tax rates)
- Manage stock via a ledger of stock movements (goods receipt, goods issue, reservation, correction). Stock levels are derived from the ledger, not stored as a single mutable number
- View and process orders (status workflow, partial shipments, cancellations, returns)
- Generate invoices (GoBD-compliant, PDF) and shipping labels (DHL)
- Design their storefront in a visual block editor, preview changes and publish them
- Receive an automatically assigned subdomain on our platform domain (e.g. `shopname.ourplatform.de`) with automatic TLS. Custom domains are explicitly **not** part of the MVP, but the design must not prevent adding them later
- Connect exactly one marketplace in the MVP (eBay): publish listings, import orders, and sync stock levels back to the marketplace. The integration must be built behind a generic marketplace abstraction so that further marketplaces (e.g. Kaufland, Amazon) can be added later as additional adapters without changing the core modules

The merchants' customers will be able to browse products, use a cart and checkout (guest checkout supported), pay, and view their order status. Customer accounts are optional.

## Multi-tenancy

- Single Postgres database, single shared schema, `tenant_id` on every tenant-owned table
- Isolation enforced via Postgres Row-Level Security (`FORCE ROW LEVEL SECURITY`, non-owner app role, tenant set per transaction via `set_config(..., true)` / `SET LOCAL`, compatible with connection pooling in transaction mode)
- A tenant registry from day one (including the tenant's subdomain and connection info), so tenants can later be moved to other DB instances (sharding / dedicated plans)
- Subdomain rules: validation, reserved names (`www`, `admin`, `api`, `auth`, etc.), uniqueness, and a policy for renaming
- Automated tests that verify tenant A can never read tenant B's data

## Architecture and Stack

- **Modular monolith** (modules: catalog, inventory, orders, invoicing, shipping, payments, storefront/pages, integrations/marketplaces), no microservices. Module boundaries should be enforced (e.g. Spring Modulith or ArchUnit)
- **Backend** (fixed, will not be changed): Kotlin with Spring Boot, Spring Security OAuth2 Resource Server, Spring Data JDBC, HikariCP. Native SQL (`@Query` / `JdbcTemplate`) for complex queries such as stock ledger aggregations. REST API with an OpenAPI spec as the contract (e.g. springdoc), webhooks, Postgres-based background job queue (e.g. JobRunr or db-scheduler). Database migrations as plain SQL with Flyway
- **RLS handling with Spring Data JDBC**: the tenant must be set at the start of every transaction (e.g. via a custom transaction manager or `DataSource` wrapper that executes `set_config('app.tenant_id', ..., true)`), never via a plain `ThreadLocal` that can leak or get lost across threads, virtual threads, async calls or background jobs. Background jobs must carry the tenant explicitly. Please propose a robust design and describe pitfalls specific to Spring Data JDBC (aggregate boundaries, repositories, no accidental queries outside a tenant-bound transaction)
- **Marketplace abstraction**: a common interface (listing publish/update/end, order import, stock sync, category and attribute mapping, error and rate-limit handling), with eBay as the first adapter. Please propose the interface design
- **Storefront**: Next.js (SSR, self-hosted via `output: 'standalone'`, shared cache handler such as Redis for multiple instances), TypeScript. The tenant is resolved from the `Host` header (subdomain). Business logic lives only in the backend, not in Next.js
- **Admin app**: Vite + React SPA with TanStack Router/Query (no SSR needed)
- **UI and styling (strict rule)**: TailwindCSS plus shadcn/ui is the only allowed UI foundation, in the admin app and in the shared `packages/blocks` storefront components.
  - Always use existing shadcn/ui components and their documented patterns.
  - Never build custom UI components or custom-styled replacements where a shadcn/ui component or composition of shadcn/ui components exists.
  - Storefront blocks (hero, product grid, footer, etc.) must be composed exclusively from shadcn/ui components and Tailwind utility classes.
  - If something seems impossible with shadcn/ui, flag it explicitly and ask me instead of building a custom solution
- **Page builder**: Puck. Pages are stored as a JSON component tree and rendered by the storefront with the same shared block components used in the editor
- **Theming**: per-tenant design tokens stored as CSS variables (colors, fonts, spacing), using the shadcn/ui theming variables, referenced by Tailwind via `@theme` and written server-side into a `<style>` tag to avoid flicker. Runtime customer changes must never be stored as Tailwind classes. No arbitrary custom JS/CSS from tenants in the MVP (XSS risk)
- **Monorepo** (npm):
  - `frontend/vgdepot-shopfront`
  - `frontend/vgdepot-admin`
  - `backend`

## Authentication and Authorization

- External identity provider: **Keycloak** (OIDC / OAuth2, Authorization Code Flow with PKCE), self-hosted with its own database
- **Merchant staff**: a single shared realm for all tenants (not one realm per tenant, for scalability). The tenant is mapped via Keycloak Organizations or a `tenant_id` claim. Roles: `owner`, `staff`
- The Spring backend (OAuth2 Resource Server) validates JWTs (signature, issuer, audience, expiry), derives `tenant_id` and role from the claims, and sets the tenant per transaction for RLS. Authorization is enforced in the backend, never in the frontend
- Password reset, email verification, and optional 2FA (TOTP) via Keycloak
- **Shop customers** (optional accounts): in the MVP managed by the backend itself (email + password or magic link, strictly scoped per tenant, so the same email can exist in several shops independently), not by Keycloak. Evaluate later whether to move them to the IdP

> **Open:** The exact Keycloak implementation is still to be clarified (Organizations vs. claim-based mapping, tenant onboarding automation via the Admin API, theming of login pages, token lifetimes, backup and upgrade strategy). Please ask me about these points and propose options with trade-offs.

## Data

- PostgreSQL as the only primary data store, with versioned migrations
- Media files in S3-compatible object storage with CDN and image resizing
- All imports, scans and webhooks must be idempotent (client-generated IDs)

## Payments (Decision Deferred)

The payment provider setup (Stripe Connect account type, PayPal variant) will be decided later and must not be specified now. **Do not ask me about it.**

- The payments module must be built behind a provider abstraction (create payment, confirm, refund, webhook handling, merchant onboarding status) so that Stripe and PayPal can be plugged in later
- Webhooks must be signed and idempotent
- Stripe and PayPal are the intended providers

## Hosting and Operations

- **Phase 1**: Hetzner (Docker, Docker Compose or Kamal/Coolify, self-managed Postgres with pgBackRest/WAL-G and regularly tested restores)
- **Later**: AWS (RDS, S3, migration via logical replication)
- Portable by design: only standard Postgres and S3 API, configuration via environment variables
- Wildcard DNS and a wildcard TLS certificate for `*.ourplatform.de` (Let's Encrypt via DNS-01 challenge, e.g. with Caddy or a DNS provider plugin). Please describe the setup for Hetzner and the later AWS migration
- JVM considerations: memory sizing, container limits, startup time. Evaluate whether GraalVM native image is worth it or whether the standard JVM with Virtual Threads (Java 21+) is sufficient

## Mobile and Desktop (Later, Not in the MVP)

- Customers of the merchants use the mobile browser only, no native shop app
- Later phase: the admin SPA packaged with Capacitor as one single app for all tenants, with barcode scanning (ML Kit plugin), offline-tolerant scan workflows and secure token storage. Desktop wrapper via Tauri is optional

## Legal and Compliance (Germany/EU)

- GoBD-compliant invoicing and archiving
- E-invoicing requirements
- VAT/OSS
- GDPR (data processing agreement, EU hosting, per-tenant deletion and export)
- Accessibility (BFSG) for storefront themes
- Legal texts/imprint/withdrawal button
- Packaging law (LUCID)

## Out of Scope for the MVP

- Custom domains
- WMS with storage locations
- POS
- Bill of materials
- Accounting/DATEV
- Additional marketplaces beyond eBay
- Native apps
- Final payment provider configuration

---

Do you need more information to create me a technical specification document which I can use as a foundation to then build this application? Please ask me about any remaining open decisions before you start (except payments, which are deferred).