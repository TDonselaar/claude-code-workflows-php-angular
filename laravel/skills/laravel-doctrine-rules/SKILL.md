---
name: laravel-doctrine-rules
description: Laravel 12 + PHP 8.3 + Doctrine ORM (SQL) and Doctrine ODM (MongoDB) hybrid persistence rules. Use when implementing Laravel features, entities/documents, service-layer code, or Doctrine repositories.
---

# Laravel + Doctrine (ORM + ODM) Rules

## Baseline

- **PHP**: 8.3+. Prefer typed properties, readonly where sensible, enums, first-class callable syntax.
- **Laravel**: 12.x. Framework features (container, routing, middleware, queues, jobs, events) are idiomatic — do not reinvent with raw PHP.
- **Doctrine**: hybrid — ORM (SQL) and ODM (MongoDB) coexist. Detect per task which one to use (see Persistence Selection below).
- **PSR-12** formatting, **PSR-4** autoloading.

## Directory Conventions

- `app/Entities/` — Doctrine entities (ORM, SQL) and Doctrine documents (ODM, MongoDB). Infer ORM vs ODM from the class' attributes (`#[ORM\Entity]` vs `#[MongoDB\Document]`), not the folder.
- `app/Services/` — domain/application services. Keep controllers thin; business logic lives here.
- `app/Adapters/` — integration points to third-party systems. Hide vendor SDK types behind adapter interfaces.
- `app/Jobs/` — queueable work. Prefer Laravel jobs over ad-hoc background scripts.
- `app/Http/Controllers/`, `app/Http/Requests/`, `app/Http/Resources/` — HTTP layer. Use FormRequest for validation and API Resources for response shaping.
- `config/doctrine.php` — ORM/ODM configuration source of truth.

## Persistence Selection (per task)

Use this decision ladder, in order, before writing persistence code:

1. **Task file or Design Doc names the persistence**: use that.
2. **Existing entity/document for this aggregate exists**: use the same mapping. Do not introduce a second persistence for the same aggregate.
3. **Domain characteristics**:
   - Transactional, relational, cross-entity invariants, reporting → ORM (SQL).
   - Document-shaped, denormalised, variable schema, high write throughput, time-series → ODM (MongoDB).
4. **When still ambiguous**: escalate. Do not pick arbitrarily — this decision drives migrations, tests, and indexes.

## Doctrine ORM (SQL) Conventions

- **Attribute mapping only** (`#[ORM\Entity]`, `#[ORM\Column]`, `#[ORM\ManyToOne]`). No XML/YAML mapping files.
- **Typed properties**. Nullable only when the database column is nullable.
- **UUIDs preferred over auto-increment** for externally-visible ids. Mark auto-increment ids as `private`.
- **Repositories**: one per aggregate root. Method names read as intent (`findActiveByOrganisation`, not `findBy(...)`).
- **Relationships**: explicit `fetch`, explicit `cascade`. Default to `fetch: LAZY`; set `cascade: ['persist']` only where the owning side is responsible for lifecycle.
- **Migrations**: generated via `php artisan doctrine:migrations:diff`; reviewed before commit. Never hand-edit a generated migration unless you also run it in a disposable database first.
- **No raw SQL in services**. Queries live in repositories using DQL or QueryBuilder. Raw SQL only for reports that cannot be expressed in DQL.

## Doctrine ODM (MongoDB) Conventions

- **Attribute mapping** (`#[MongoDB\Document]`, `#[MongoDB\Field]`, `#[MongoDB\ReferenceOne]`, `#[MongoDB\EmbedOne]`). No XML.
- **Embed vs reference**: embed for lifecycle-bound sub-documents (always read/written together, < ~1MB, not shared); reference when the sub-document is shared or lives independently.
- **Indexes declared on the document** via `#[MongoDB\Index]`. A query pattern without an index is a design defect — either add the index or change the query.
- **No queries against `_id` as a string** — cast via `MongoId` / `ObjectId` at the boundary.
- **Schema migrations** via a deliberate migration job (not implicit on-write). Document any backfill in the Design Doc.
- **Pagination** uses `skip`/`limit` only for small result sets; use keyset pagination for anything user-visible with >1000 rows.

## Service Layer

- Services are invoked by controllers and jobs. They accept validated input (DTOs, FormRequest data, primitives) — never raw `$request`.
- Throw **domain exceptions**, let Laravel's exception handler map them to HTTP responses via the `render` method in `Handler.php`.
- **No static state**. Resolve collaborators via constructor injection. Facades are allowed only in controllers/jobs for Laravel primitives (Log, DB, Queue).
- **Transactions**:
  - ORM: wrap multi-step writes in `EntityManager::transactional(...)`.
  - ODM: MongoDB transactions require a replica set. Prefer idempotent design and compensation over distributed transactions; when you do use sessions, keep them short.

## HTTP Layer

- **Routes**: `routes/api.php` for JSON, `routes/web.php` for session-backed UI. Group by domain, version with URI prefix (`/api/v1/...`) when breaking changes ship.
- **Controllers**: action-per-class (`__invoke`) preferred for non-trivial endpoints.
- **FormRequest** for input validation. Authorisation lives in `authorize()` and uses policies.
- **API Resources** shape responses. Never return Eloquent/Doctrine entities directly from a controller.

## Error Handling

- Domain errors: typed exceptions in `app/Exceptions/Domain/`. Caught at the HTTP boundary and mapped to problem-details JSON.
- Infrastructure errors: wrap vendor-specific exceptions in adapter-level exceptions so services never catch vendor types.
- **Every caught `\Throwable`** is either rethrown, converted to a domain exception, or logged with `report($e)` and handled. Silent catches are forbidden.

## Logging

- Use Laravel's `Log` facade or an injected `LoggerInterface`. Include structured context (arrays) — do not interpolate user input into the message.
- Redact: passwords, tokens, secrets, API keys, full credit card numbers, long opaque blobs.

## Queues & Jobs

- Any operation > ~500ms or with external IO belongs in a queued job.
- Jobs are **idempotent** — assume at-least-once delivery. Guard on an idempotency key or an authoritative state check.
- Define `$tries`, `$backoff`, `$timeout` explicitly. Do not rely on framework defaults in production code.

## Type Safety

- No `mixed` in public signatures without a comment explaining why.
- No `@var` docblocks to compensate for missing types — widen the real type or introduce a value object.
- Enums for closed sets. String unions only when interop with an external system demands them.

## Tests

See `pest-testing-rules` skill for the test discipline. Every new service method has a Feature test (through the framework) or a Unit test (pure logic). Doctrine boundaries are exercised by Feature tests against the real in-memory/containerised database, not mocked repositories.

## Security

- **Mass assignment**: never pass unvalidated input into `fill()`. Use FormRequest `validated()` arrays.
- **SQL injection**: DQL/QueryBuilder with parameters; never string-concatenate query fragments.
- **NoSQL injection**: never place user input directly into a criteria array without type casting (`(string)`, `(int)`, ObjectId).
- **Authorisation**: Policies + `authorize()` in FormRequest. No authorisation decisions in Blade templates.
- **Secrets**: `.env`, never committed. Use `config('services.x.key')` — never `env()` at call sites outside of config files.

## Comments

Default to none. Add a short line only when the **why** is non-obvious (business invariant, known caveat, external-system quirk). Do not annotate the what.
