## Project Overview
Multi-storefront e-commerce API (**GraphQL-first**) built with NestJS and Fastify. Covers vendors, vendor facets, catalog, search, carts, checkout (tax, coupons, payments, idempotency), draft orders, shipments, inventory, CMS/metaobjects, marketing subscribers, analytics, AI recommendations, PDF export, and a hierarchical agencies model (agency → bureau → office → group, with user assignments). REST controllers remain in the codebase for DI and tests, but **HTTP access to `/api/*` is disabled** at the edge in favor of GraphQL.

## Core Technology Stack
- **Framework**: NestJS 11 with Fastify adapter (HTTP/2 enabled)
- **Primary API**: GraphQL (Apollo Server 5, `@nestjs/graphql` code-first) at `POST /graphql`
- **REST**: Versioning is configured (`/api/v1/...`) but routes return **404** by design (see `main.ts`)
- **Language**: TypeScript (strict mode)
- **Database**: PostgreSQL with TypeORM (`synchronize: false`)
- **Scheduling**: `@nestjs/schedule` (`ScheduleModule.forRoot()` in `AppModule`); see `commerce/commerce-cron.service.ts`
- **Error monitoring**: Sentry via `@sentry/nestjs` — bootstrapped in `src/instrument.ts` (imported at the top of `main.ts`), wired as a global filter in `AppModule`
- **Storage**: Google Cloud Storage (`@google-cloud/storage`)
- **Payments**: Stripe (`stripe` package) plus manual provider abstraction
- **Auth**: JWT with Passport; `bcrypt` for password hashing; roles `ADMIN`, `VENDOR`, `USER` (`auth/enums/role.enum.ts`)
- **Other notable libs**: `puppeteer` (PDF/HTML rendering), `sharp` (images), `axios`, `sanitize-html`, `nestjs-typeorm-paginate`, `graphql-type-json`, `nanoid`
- **Logging**: Fastify/Pino (pretty transport in development via `pino-pretty`); custom Apollo plugin logs GraphQL operation + query when `NODE_ENV !== 'production'`

---

## Current Project Structure

```
src/
├── instrument.ts                # Sentry.init — must be imported FIRST (top of main.ts)
├── main.ts
├── app.module.ts
├── app.controller.ts / app.service.ts
├── auth/
│   ├── auth.controller.ts       # present on disk; NOT registered in AuthModule (GraphQL only)
│   ├── auth.resolver.ts
│   ├── auth.service.ts
│   ├── decorators/              # @CurrentUser, @Roles
│   ├── dto/, entities/, enums/  # entities: UserVerification, UserLogin, PasswordReset; enum: Role
│   ├── guards/, strategies/, types/
├── users/
│   ├── addresses/
│   ├── dto/, entities/, enums/, types/
├── agencies/                    # hierarchical: Agency → Bureau → Office → Group
│   ├── agencies.module.ts
│   ├── agencies.service.ts / .resolver.ts
│   ├── agency-hierarchy.resolver.ts
│   ├── bureaus.service.ts / offices.service.ts / groups.service.ts
│   ├── user-agency-assignments.service.ts
│   ├── dto/, entities/          # Agency, Bureau, Office, Group, UserAgencyAssignment
├── vendors/
│   ├── contact-requests/
│   ├── decorators/              # @VendorScoped, @CurrentVendor
│   ├── guards/                  # VendorScopedGuard
│   ├── entities/                # Vendor, VendorMembership, VendorShippingMethod
│   └── vendor-facets/           # VendorFacet, VendorFacetValue (+ resolver/service)
├── commerce/                    # Aggregator: commerce.module.ts (19 submodules + CommerceCronService)
│   ├── commerce.module.ts
│   ├── commerce-cron.service.ts # hourly jobs (inventory + cart cleanup)
│   ├── categories/
│   ├── products/
│   ├── collections/
│   ├── facets/
│   ├── tags/
│   ├── reviews/
│   ├── inventory/
│   ├── carts/
│   ├── orders/
│   ├── shipments/
│   ├── quotes/
│   ├── demo-requests/
│   ├── favorites-lists/
│   ├── search/                  # commerce search sync entity + jobs
│   ├── tax/
│   ├── coupons/
│   ├── draft-orders/
│   ├── payments/                # Stripe/manual providers, storefront payment methods
│   └── checkout/                # cart/draft checkout, idempotency records
├── storefronts/
│   ├── content/, menus/, pages/, vendors/  # vendors/ = storefront↔vendor ordering
├── metaobjects/
│   ├── controllers/, resolvers/, services/, entities/, enums/
├── contents/                    # global Content entity (CMS-style)
├── storage/                     # Global module — inject StorageService anywhere
├── marketing/
│   └── subscribers/
├── analytics/
├── customizations/
│   └── ai-recommendations/
├── exports/
│   └── pdf/                     # GraphQL-driven PDF generation
├── database/
│   ├── data-source.ts           # TypeORM CLI + Cloud SQL socket path when DB_HOST starts with /cloudsql/
│   └── migrations/              # 91 migration files (and growing)
└── schema.gql                   # Auto-generated GraphQL schema
```

---

## Architecture & Design Principles

### 1. Module Organization
- Feature modules live as **top-level domains** under `src/` (not `src/modules/`)
- **`CommerceModule`** aggregates **19** feature modules (see tree above); also registers `CommerceCronService` for scheduled jobs
- Prefer the standard Nest shape per feature: `*.module.ts`, `*.service.ts`, optional `*.controller.ts`, optional `*.resolver.ts`, plus `dto/`, `entities/`, `enums/`, `types/` as needed
- Keep modules loosely coupled; consume **exported** providers from other modules
- **`StorageModule` is `@Global()`** — inject `StorageService` without importing the module everywhere

### 2. Multi-Storefront Architecture
- Many resources are storefront-scoped (categories, collections, pages, menus, storefront content, metaobjects, tax, payment methods, checkout idempotency, etc.)
- Users relate to storefronts (many-to-many patterns in migrations/entities)
- Vendors link to storefronts via `storefronts/vendors/` with ordering

### 3. Vendor Scoping
- Vendor-owned rows carry `vendorId` where applicable
- Use `@VendorScoped` + `VendorScopedGuard` for ownership checks
- Use `@CurrentVendor` to read the active vendor from request context
- Orders support parent/suborder splitting per vendor
- Vendor taxonomy uses `vendor-facets/` (separate from product `facets/`)

### 4. Agencies / Org Hierarchy
- Four-level hierarchy: **Agency → Bureau → Office → Group**
- Users attach to any level via `UserAgencyAssignment`
- Read flows via `AgenciesResolver` + `AgencyHierarchyResolver`; write flows via per-level services

### 5. Dependency Injection
- Constructor injection only; `@Injectable()` services — avoid `new` for providers

### 6. DTOs and Validation
- Separate Create/Update/Filter DTOs; GraphQL uses `@InputType()` / field types where needed
- `class-validator` + `class-transformer`
- Global `ValidationPipe`: `whitelist`, `forbidNonWhitelisted`, `transform`, custom `exceptionFactory` for field-level messages

```typescript
// GraphQL example — use Role enum, not raw strings
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN)
@Mutation(() => TaxRate)
createTaxRate(@Args('input') input: CreateTaxRateInput) { }
```

---

## Bootstrap Configuration (main.ts)

- **First line imports `./instrument.js`** — Sentry must initialize before NestFactory
- **Fastify** with HTTP/2, **10MB** body limit, **64KB** max header size (custom `serverFactory`)
- Early **hook**: any path `/api` or `/api/*` → **404** with message to use GraphQL (`POST /graphql`)
- Dev-only `onSend` hook logs response payloads via the Fastify logger
- **Compression** (`@fastify/compress`), **Helmet** (CSP tuned for Apollo playground + admin/vendor/storefront origins)
- **Multipart** uploads, **5MB** per file
- **Global rate limit**: 10,000 / 10 minutes (stricter per-route auth limits are **commented out** — revisit if REST is re-enabled)
- **Global prefix** `api` with exclusions for `/` and `assets/*`
- **URI versioning** `v1` still registered for controller metadata
- **CORS**: localhost admin/vendor/storefront ports + production `*.gcommerce.glass` hosts and Cloud Run host
- **Listen**: `0.0.0.0`, port `8080` or `PORT`

---

## GraphQL Implementation

### App wiring (`app.module.ts`)
- Apollo driver, `autoSchemaFile`: `src/schema.gql`, `sortSchema: true`
- Playground + introspection when `NODE_ENV !== 'production'`
- **`csrfPrevention: true`** — clients must send `Content-Type: application/json` (non-empty) or the `Apollo-Require-Preflight` header
- **Dev plugin** logs operation name, query, and variables on `didResolveOperation` (disabled in production)
- **Context** exposes `req` (with `ctx.request` fallback) and `reply` so JWT/`Roles` guards work on GraphQL
- **Global `SentryGlobalFilter`** registered via `APP_FILTER` — uncaught GraphQL/REST errors are reported

### Practices
- Thin resolvers; business logic in services
- DataLoader where N+1 is a risk
- Field resolvers for computed / joined data
- Connection types for pagination (e.g. products, reviews, subscribers)

---

## REST API (current state)

- Controllers and DTOs exist for many domains, but **HTTP requests under `/api` are rejected** in `main.ts`
- Some modules no longer register their controller (e.g. `AuthModule` only wires `AuthResolver`); the controller file remains on disk
- Intended public surface today: **GraphQL only**
- OpenAPI/Swagger is **not** wired in `main.ts`

---

## Scheduled Jobs

- `ScheduleModule.forRoot()` is registered in `AppModule`
- `commerce/commerce-cron.service.ts` runs hourly via `@Cron(CronExpression.EVERY_HOUR)`:
  - `cleanupExpiredInventoryReservations` — releases reservations past their hold window
  - `markExpiredCartsAbandoned` — flips `ACTIVE` carts past `expiresAt` to `ABANDONED`
- Failures are logged but not rethrown (jobs are best-effort)

---

## Database

### TypeORM + PostgreSQL
- Entities: glob `**/*.entity{.ts,.js}`
- **`synchronize: false`** — schema changes via migrations only
- **`migrationsRun: true` in production** when `NODE_ENV === 'production'` (see `AppModule` TypeORM config) — deploy pipeline expects this
- `data-source.ts` supports Cloud SQL Unix socket via `DB_HOST` starting with `/cloudsql/` (passed as `socketPath` in `extra`)

### Migrations
- **91** TypeScript files under `src/database/migrations/` (count increases with schema work)
- Generate: `npm run migration:generate --name=MigrationName`
- Empty migration: `npm run migration:create --name=MigrationName`
- Run / revert / show: `npm run migration:run` | `migration:revert` | `migration:show`
- Production CLI: `npm run migration:run:prod` (compiled `dist/`)

### Entity conventions
- UUID PKs (`@PrimaryGeneratedColumn('uuid')`) where used
- `@CreateDateColumn` / `@UpdateDateColumn`; soft deletes with `@DeleteDateColumn` where modeled
- `vendorId` / `storefrontId` on scoped entities as applicable
- JSONB for flexible payloads (orders, metaobjects, etc.)
- Trees via `parentId` patterns (categories, menus, pages, contents) and via dedicated hierarchy entities (agencies)

### Entity inventory (**60** `.entity.ts` files)
- **Auth / users**: User, Address, UserVerification, UserLogin, PasswordReset
- **Agencies**: Agency, Bureau, Office, Group, UserAgencyAssignment
- **Vendors**: Vendor, VendorMembership, VendorShippingMethod, VendorContactRequest, VendorFacet, VendorFacetValue
- **Storefronts**: Storefront, Menu, Page, Content (storefront), StorefrontVendor
- **Commerce**: Product, ProductVariant, ProductImage, ProductDocument, Category, Collection, CollectionRule, Cart, CartItem, CartShippingSelection, Order, OrderItem, Shipment, Inventory, InventoryReservation, InventoryHistory, Facet, FacetValue, ProductTag, ProductReview, Quote, DemoRequest, FavoritesList, FavoritesListItem, CommerceSearchSync, TaxRate, Coupon, CouponUsage, DraftOrder, DraftOrderItem, DraftOrderShipping, Payment, StorefrontPaymentMethod, CheckoutIdempotency
- **CMS / dynamic**: Content (global module), MetaobjectDefinition, MetaobjectEntry
- **Marketing / customizations**: Subscriber, AiRecommendation

---

## Security

- **JWT** + **Passport** (`JwtStrategy`, `JwtAuthGuard`, `RolesGuard`, `VendorScopedGuard`)
- Password hashing with **bcrypt**
- **Decorators**: `@CurrentUser()`, `@Roles(Role.ADMIN | ...)`, `@VendorScoped()`, `@CurrentVendor()`
- **ValidationPipe** hardening; HTML via `sanitize-html` where appropriate
- **Helmet** + global **rate limiting** (see main.ts)
- **Vendor membership** enforces multi-vendor access patterns
- Sentry captures unhandled exceptions globally (`SentryGlobalFilter`) — avoid sending PII in error messages or context

---

## Observability

- **Sentry**: `SENTRY_DSN` enables capture (off by default if unset); `sendDefaultPii: true` in `instrument.ts`
- **Logs**: Pino (pretty in dev) — request method/URL formatted via `messageFormat`
- **GraphQL dev plugin**: logs operation + query + variables in non-production

---

## Testing

- Jest + ts-jest; tests under `src/**/*.spec.ts`
- E2E: `test/jest-e2e.json`
- Commands: `npm test`, `npm run test:watch`, `npm run test:cov`, `npm run test:e2e`
- Prefer AAA structure; mock external IO (Stripe, GCS, Puppeteer, Sentry, etc.)

---

## Environment & Configuration

- Global `ConfigModule` (`.env`, not committed)
- Typical keys: `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `PORT`, `NODE_ENV`, `SENTRY_DSN`, GCS settings, Stripe secrets as used by payments module
- Scripts: `npm run start:dev`, `npm run build`, `npm run start:prod`, `npm run lint`, `npm run format`

---

## Code Quality Standards

- TypeScript strict, ESLint 9 + Prettier
- Small functions, shallow nesting, avoid dead commented code
- Services own business rules; controllers/resolvers stay thin

---

## Planned Features

### Notifications Module (planning)
- Spec: `.docs/02-Planning/notifications-module.md`
- Transactional email, storefront-scoped templates/settings, event triggers, ADMIN management

---

## Common Pitfalls to Avoid

1. **Don't** assume REST is callable — use GraphQL unless the `/api` hook is removed
2. **Don't** bypass `ValidationPipe` or rely on unvalidated `any` payloads
3. **Don't** block the event loop on heavy sync IO
4. **Don't** leak stack traces / internal errors to clients (Sentry captures, clients should not)
5. **Don't** add vendor/storefront rows without scoping rules and guards
6. **Don't** put core business logic in resolvers/controllers
7. **Don't** ignore strict TypeScript errors
8. **Don't** change the DB without a migration
9. **Don't** fetch unbounded relation graphs without pagination or DataLoader discipline
10. **Don't** import anything before `./instrument.js` in `main.ts` — Sentry must initialize first
11. **Don't** register a `@Cron` job without idempotent / try-catch behavior — single-instance assumption is not guaranteed

---

## Additional Resources

- [NestJS Documentation](https://docs.nestjs.com)
- [Fastify Documentation](https://www.fastify.io)
- [TypeORM Documentation](https://typeorm.io)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices)
- [Sentry NestJS SDK](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [OWASP Security Guidelines](https://owasp.org)
- In-repo skills: `.cursor/skills/graphql-api-designer/SKILL.md`, `.cursor/skills/schema-architect/SKILL.md`, `.cursor/skills/security-review/SKILL.md`
- Lifecycle documentation index: `.docs/README.md`
