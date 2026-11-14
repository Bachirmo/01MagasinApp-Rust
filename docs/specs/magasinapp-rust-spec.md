# MagasinApp — Rust + SQLite Architecture Specification

**Status:** Draft for review  
**Target:** Windows desktop (Tauri 2)  
**Supersedes:** Legacy implementation and deployment decisions only where they conflict with this document.

## 1. Purpose and governing principle

MagasinApp is being reimplemented with a different backend and desktop runtime while preserving the established product behavior and user experience.

The legacy repository remains the source of truth for functional requirements, business rules, interface structure, labels, layout, and interactions. This repository implements those requirements using a Rust-native backend and a local SQLite database.

**The intended change is the technology underneath the application—not a redesign or a reduction of functionality.**

## 2. Goals

- Preserve the existing Nuxt/Vue interface, navigation, dashboard, Caisse, catalog, tables, dialogs, and interaction patterns.
- Preserve the behavior of products, categories, favorites, users, sales, stock, reports, imports, exports, backups, and restore.
- Replace Nitro HTTP API, Prisma, and PostgreSQL with Rust, SQLx, and SQLite.
- Package the application for Windows using Tauri 2.
- Run business logic and database access in the native Rust application process.
- Work locally after installation without Internet access.
- Keep business records durable across application upgrades.
- Keep the legacy repository unchanged.

## 3. Non-goals

- Redesigning the interface or changing established workflows.
- Introducing PocketBase.
- Running Nitro, Node.js, or another HTTP backend as a sidecar.
- Requiring a separately installed database server.
- Using PostgreSQL or Prisma in the new application runtime.
- LAN hosting, multi-PC shared database access, cloud synchronization, or queued offline sales.
- Smartphone, macOS, or Linux releases in this initiative.
- Changing business rules merely to fit a new storage engine.

## 4. Technology decisions

| Concern | Decision |
| --- | --- |
| Desktop host | Tauri 2 |
| Frontend | Existing Nuxt 4 / Vue interface, packaged as local static assets |
| UI components | Preserve existing Nuxt UI and Tailwind CSS implementation |
| Frontend rendering | Static/client-rendered build; no Nitro server at runtime |
| Native backend | Rust in the Tauri application process |
| Frontend/backend boundary | Tauri commands and typed IPC |
| Database | SQLite, one local database per installation |
| Database access | SQLx |
| Schema evolution | Versioned SQLx migrations |
| Serialization | Serde |
| Runtime network API | None for application-internal operations |
| Sidecar processes | None |

Rust is authoritative for business rules, authorization, and persistence. The frontend must not access the SQLite file directly.

## 5. High-level architecture

~~~mermaid
flowchart TD
    UI["Nuxt 4 / Vue UI<br/>Static packaged assets"]
    IPC["Tauri IPC<br/>Typed commands and events"]
    APP["Rust application layer"]
    AUTH["Authentication and authorization"]
    DOMAIN["Domain services<br/>Sales, stock, catalog, reports"]
    DB["SQLx repositories"]
    SQLITE[("SQLite database")]
    FS["Application data<br/>Backups, exports, logs"]

    UI <--> IPC
    IPC <--> APP
    APP --> AUTH
    APP --> DOMAIN
    DOMAIN --> DB
    DB <--> SQLITE
    APP --> FS
~~~

### 5.1 Frontend

Keep the existing Nuxt/Vue interface as the presentation layer and package its built assets inside Tauri. Replace calls to Nitro HTTP endpoints with a typed frontend service layer that invokes Tauri commands.

Pages and components should call frontend services rather than invoking Tauri commands ad hoc. This keeps the UI independent of command naming and makes the backend boundary explicit.

The frontend must not contain database credentials, password hashes, PIN hashes, session secrets, or direct SQL access.

### 5.2 Rust application layer

Organize Rust code by responsibility, with narrow interfaces between command handlers, application services, domain rules, and persistence.

Suggested initial structure, to be refined during implementation:

~~~text
src-tauri/
  src/
    main.rs
    lib.rs
    commands/
    application/
    domain/
    auth/
    db/
      migrations/
      repositories/
    infrastructure/
    errors/
  Cargo.toml
~~~

- **Commands:** IPC boundary; deserialize inputs, invoke application services, and serialize safe outputs.
- **Application services:** Coordinate use cases and transactions.
- **Domain:** Business rules and domain types; avoid Tauri and SQLx dependencies where practical.
- **Repositories:** SQLx queries and row mapping.
- **Infrastructure:** File paths, backup/restore, export/import, logging, and OS integration.
- **Auth:** Password/PIN verification, session lifecycle, role checks, and idle-lock support.

Do not expose raw database rows as the public command contract. Return explicit DTOs.

### 5.3 IPC contract

- Use Tauri commands for request/response operations.
- Use Tauri events only for genuine asynchronous notifications.
- Define stable, typed request and response DTOs.
- Return structured, user-safe errors with stable error codes.
- Enforce authorization in Rust for every protected command. Hiding a UI action is not authorization.
- Do not expose arbitrary SQL execution or generic file-system commands to the frontend.
- Keep command payloads bounded and validate identifiers, quantities, dates, and text lengths.

## 6. Functional compatibility requirements

The legacy functional specification and interface document remain authoritative. Preserve their requirements unless a change is explicitly approved.

### 6.1 Interface and localization

- Preserve existing page structure, navigation, dashboard, Caisse layout, product tiles, tables, dialogs, and interaction patterns.
- Preserve Arabic as the default language and French as the alternative.
- Preserve RTL/LTR behavior, Western digits, DD/MM/YYYY dates, and existing translations.
- Preserve laptop and tablet support and touch targets of at least 44 px.
- Do not introduce a visual redesign as part of backend migration.

### 6.2 Roles and access

Preserve the existing roles and permission boundaries:

- **Admin:** sole account-creation authority and full administrative access.
- **Delegated manager:** operational administrative access, with restrictions documented in the legacy specification.
- **Cashier:** PIN sign-in and Caisse-only access.

Rust must enforce role and account-status checks on every protected command. Preserve first-run setup, password/PIN rules, account deactivation behavior, and cashier idle-lock behavior.

### 6.3 Catalog and inventory

Preserve bilingual category and product names, category colors, references and barcodes, archive/reactivation behavior, favorite ordering, stock quantities, and restock reasons.

Every stock-changing operation must create its corresponding stock movement and audit information in the same database transaction as the stock update.

### 6.4 Caisse and sales

Preserve the 15 favorite slots, full-catalog search, cart behavior, payment flow, invoice details, and existing stock-warning/checkout rules.

A completed sale must be atomic. At minimum, one transaction must coordinate:

1. Sale header creation and sale-number allocation.
2. Sale-line creation with French and Arabic product-name snapshots and unit-price snapshots.
3. Stock updates according to the approved business rules.
4. Stock movement records.
5. Relevant audit records.

A failure at any step must roll back the whole transaction. Sale numbering must remain unique and correctly scoped to the business year, following the existing functional specification.

Low stock must not become a new checkout blocker where the legacy requirements explicitly allow checkout. Do not silently change stock policy during migration.

Sale cancellation must remain an administrative operation with a required reason, stock restoration, audit details, and exclusion from revenue totals.

### 6.5 Dashboard, reports, imports, and exports

Preserve existing dashboard periods and metrics, sales history and filters, rankings, imports, CSV exports, Excel workbook contents, and row-level import errors.

Exports must never include password or PIN hashes. File generation and parsing must run locally.

### 6.6 Failure and outage behavior

If the local database is unavailable or an operation fails:

- Keep an unsubmitted cart in the UI.
- Do not queue sales for later.
- Do not automatically retry or resubmit a sale.
- Show a clear error and require an explicit user retry.
- Never show a success invoice until the sale transaction has committed successfully.

## 7. SQLite and SQLx data design

### 7.1 Schema migration

Translate the existing Prisma data model and its constraints into explicit SQLite DDL and SQLx migrations. Preserve entity relationships, uniqueness, indexes, nullability, and business invariants.

Expected domain records include users, categories, products, app settings, sales, sale lines, annual sale counters, stock movements, and audit records. Confirm the final schema against the complete legacy Prisma schema before implementation; this list is not a substitute for that review.

### 7.2 SQLite-specific rules

- Enable and verify foreign-key enforcement on every database connection.
- Use parameterized SQL through SQLx.
- Store money as integer Moroccan Dirham centimes; do not use floating-point money.
- Store timestamps in one documented, consistently parsed representation (UTC instants with explicit local-time presentation rules).
- Use explicit constraints and indexes for uniqueness and query performance.
- Use SQLite transactions for every multi-record business operation.
- Evaluate WAL mode for normal local operation and document checkpoint/backup behavior.
- Configure a bounded busy timeout and handle SQLITE_BUSY with a safe, user-visible failure path. Never duplicate a sale automatically.
- Keep write transactions short. SQLite supports concurrent readers but serializes writers; do not design for simultaneous multi-PC writers.
- Use SQLite-supported JSON text/functions only if a concrete requirement needs them. PostgreSQL JSONB is not available and must not be assumed.
- Do not depend on PostgreSQL-specific types, operators, extensions, or SQL syntax.

SQLx and SQLite support transactions. Use them for checkout, restocking, sale cancellation, imports that update related records, and other atomic multi-record operations. Transaction boundaries belong in application services/use cases, not UI components. Repository methods should participate in the caller's transaction where an operation spans multiple repositories.

## 8. Authentication and security

- Hash passwords and cashier PINs using a suitable password-hashing algorithm with per-credential salts and parameters selected for the threat model. Never store plaintext credentials.
- Keep authentication verification and authorization in Rust.
- Do not return hashes or secrets in DTOs, logs, diagnostics, or exports.
- Protect the local database and backup files with Windows user-scoped filesystem permissions.
- Use operating-system secure storage for any application secret that must persist; do not hardcode secrets in source code or frontend assets.
- Restrict Tauri capabilities and command exposure to the minimum required by the UI.
- Validate all inputs again in Rust, even if the frontend already validates them.
- Treat imported files and restored backups as untrusted input; validate before applying them.
- Avoid logging sensitive business or authentication data unnecessarily.

Document the limits of local-at-rest protection: a person with sufficient access to the Windows account or machine may be able to access that user's application data.

## 9. Windows runtime and application data

- Target Windows desktop with Tauri 2.
- Store the SQLite database, backups, logs, and mutable configuration in a durable per-user application-data directory, not in the installation directory.
- Resolve paths using Tauri/OS application-data APIs; do not hardcode a developer-specific path.
- Ensure upgrades do not overwrite or delete user data.
- Provide clear first-run initialization and migration.
- Keep logs useful for diagnosis but redact credentials, PINs, hashes, and secrets.
- No Nitro process, Node.js process, HTTP loopback server, or sidecar is required.

## 10. Backup, restore, and data migration

### 10.1 SQLite backup and restore

Implement consistent backups using SQLite-supported backup mechanisms or another method proven safe for the chosen journal mode. Do not blindly copy a live database file if that could omit committed WAL data.

Preserve the legacy requirements for daily/on-demand backups, retention of up to 30 archives, explicit restore warning, and typed confirmation. Validate a backup before replacing the active database, and keep a recoverable pre-restore copy.

### 10.2 PostgreSQL-to-SQLite migration

Existing PostgreSQL records may need to be retained. Treat migration as a separate, explicit, one-time data migration—not runtime compatibility.

Before switching the shop to the new app:

1. Take and preserve a source PostgreSQL backup.
2. Read from a selected source database without modifying it.
3. Map records into the SQLite schema with explicit handling for types, IDs, timestamps, money, constraints, and sequences.
4. Import into a new SQLite database in controlled transactions.
5. Compare record counts and validate representative users, roles, categories, products, stock, sales, sale lines, cancellations, stock movements, and audit records.
6. Verify important totals and relationships.
7. Produce a migration report and retain both source and destination backups.
8. Switch only after the owner explicitly accepts the verification results.

Never silently discard records or modify the legacy PostgreSQL database. The migration tool must be repeatable against a fresh destination and fail safely on invalid or ambiguous data.

## 11. Testing and quality gates

Test behavior at several layers:

- **Domain tests:** role permissions, pricing, sale numbering, stock rules, cancellation, date-period calculations, and import matching.
- **Database integration tests:** SQLx migrations, constraints, foreign keys, transaction rollback, uniqueness, and repository behavior against temporary SQLite databases.
- **Command tests:** DTO validation, authorization, safe errors, and command-to-service behavior.
- **Frontend integration tests:** existing UI workflows against the Tauri command boundary.
- **End-to-end tests:** first-run setup, login, Caisse checkout, stock restock, cancellation, dashboard, imports/exports, backup, and restore.
- **Windows packaged tests:** clean install, first launch, upgrade while preserving data, offline operation, and recovery after database errors.

Critical acceptance criteria:

- A failed checkout leaves no partial sale, sale lines, stock updates, or stock movements.
- A successful checkout records all required snapshots and updates atomically.
- A failed or interrupted restore does not destroy the only usable database.
- No command allows a cashier to invoke administrative operations.
- No sale is queued or automatically submitted after a failure.
- Arabic/French layouts and established interface behavior remain unchanged.
- The packaged application works without Internet after installation.

## 12. Migration strategy and implementation sequence

Work in small, reviewable pull requests in this repository. Keep the legacy repository unchanged.

1. **Architecture and contract:** approve this document; inventory legacy pages, API operations, Prisma schema, and tests; define frontend service and Rust command contracts.
2. **Frontend extraction:** package the existing Nuxt interface with minimal changes; replace backend calls behind a service boundary without changing user-facing behavior.
3. **Tauri/Rust foundation:** create the Tauri 2 shell, Rust application structure, command registration, typed errors, and Windows development/build workflow.
4. **SQLite foundation:** add SQLx, connection configuration, migrations, test database helpers, and schema constraints.
5. **Authentication and authorization:** implement first-run setup, password/PIN verification, roles, session/idle-lock behavior, and command guards.
6. **Catalog and inventory:** implement categories, products, favorites, restocking, and stock movements.
7. **Caisse and sales:** implement catalog lookup, cart-facing commands, atomic checkout, sale numbering, snapshots, and cancellation.
8. **Dashboard and data tools:** implement reports, imports, exports, backup, and restore.
9. **Data migration:** build and verify the optional PostgreSQL-to-SQLite migration utility.
10. **Release hardening:** complete Windows offline, security, recovery, upgrade, and installer acceptance tests.

Each step must have its own acceptance criteria and tests. Do not begin a dependent feature until its required foundation is verified.

## 13. Source documents and precedence

The following files in the legacy repository remain the functional and visual reference:

- docs/specs/magasinapp-spec.md — functional requirements and business rules.
- docs/specs/magasinapp-tickets.md — original implementation breakdown and dependencies.
- docs/specs/user-interface.md — interface structure, visual behavior, and interaction details.

The legacy Windows deployment specification (#29) and ADR 0001 describe the previous Nitro-sidecar/PostgreSQL approach. Their deployment decisions are superseded for this new implementation wherever they conflict with this document. Their functional requirements remain useful only where compatible with the new architecture.

If this document conflicts with an established functional or UI requirement, do not silently choose one: record the conflict and resolve it explicitly before implementation.

## 14. Open decisions before implementation

The following details must be confirmed during the first implementation tickets:

- Exact legacy Prisma schema, indexes, constraints, and all API operations to map.
- Final SQLx migration layout and database connection lifecycle.
- Password-hashing crate and parameters, plus Windows secure-storage choice for any persistent secret.
- SQLite backup API/library and WAL policy.
- Exact date/time and business-year rules used by existing sale numbers and reports.
- Whether the first release needs a migration utility bundled with the app or a separate one-time tool.
- Tauri capability allowlist and Windows installer/signing approach.

These are implementation decisions; they do not reopen the agreed core choices: Rust backend, SQLx, SQLite, Tauri 2, no HTTP sidecar, and preservation of the existing interface and functional behavior.
