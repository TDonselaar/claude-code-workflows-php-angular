---
name: pest-testing-rules
description: Pest 3 + Laravel testing rules. Use when writing or modifying tests for a Laravel project that uses Pest (not PHPUnit directly).
---

# Pest Testing Rules (Laravel 12)

## Runner

- `composer test` is the canonical entry point (wraps `artisan test`).
- Feature tests live in `tests/Feature/`; unit tests in `tests/Unit/`.
- `phpunit.xml` controls env overrides for the test run (in-memory SQLite for ORM, test-scoped Mongo database for ODM).

## Style

- Use Pest's functional syntax (`it()`, `test()`, `expect()`). Do not mix PHPUnit class-style tests in the same file.
- One behaviour per `it()`. Name it as a full sentence: `it('rejects a payload without an idempotency key')`.
- **Arrange-Act-Assert** structure with blank lines between sections when a test runs more than ~6 lines.

## Test Selection

| What changed | Test type | Location |
|---|---|---|
| Pure logic in a service / value object | Unit | `tests/Unit/` |
| HTTP endpoint, queue job, event listener, Doctrine repository | Feature | `tests/Feature/` |
| Cross-aggregate invariant or multi-step flow | Feature | `tests/Feature/` |
| Console command | Feature | `tests/Feature/Console/` |

Never stub the database or Doctrine layer in a Feature test. If the test would need a mocked repository, it belongs in `tests/Unit/` and the thing under test is wrong.

## Datasets & Higher-Order Tests

- Use `dataset()` for exhaustive input/output tables. Prefer named datasets (`dataset('valid emails', [...])`) over anonymous arrays — the failure messages are far better.
- Use higher-order tests only when the chain reads naturally in English (`it('...')->assertStatus(200)`). Nested expectations > higher-order chains for anything with more than one assertion.

## Architecture Tests

- Add Pest architecture tests for boundaries that matter: no `use` of vendor SDKs outside `app/Adapters/`, no `dd()`/`dump()` in shipped code, controllers do not inject repositories directly.
- Keep them fast — one `arch()` file per invariant.

## Fixtures

- **Factories over fixtures**. Entity/document factories live next to the class or under `database/factories/` for ORM (Laravel's default location) and `tests/Factories/` for ODM documents.
- Seed only the minimum required for the assertion. Do not reuse giant "world setup" functions across unrelated tests — they hide coupling.

## Doctrine in Tests

- **ORM**: use an in-memory SQLite database when possible; otherwise a throwaway database-per-process. `RefreshDatabase`-equivalent is `->refreshDatabase()` via `uses()` on the test file or the global `Pest.php`.
- **ODM**: use a dedicated `laravel_test` Mongo database. Drop collections between tests, not whole databases (faster).
- Clear the identity map between tests that share a connection — stale entities hiding state bugs is a common failure mode.

## Assertions

- Prefer specific assertions: `->toHaveCount(3)`, `->toMatchArray(...)`, `->toThrow(DomainException::class)`.
- For JSON responses, use `$response->assertJsonPath('data.0.id', $id)`; avoid comparing whole response bodies when a single field is the behaviour under test.
- Negative assertions (`->not->toContain(...)`) are fine but must be paired with a positive assertion that proves the test was meaningful.

## Test Doubles

- Use Laravel's `Event::fake()`, `Queue::fake()`, `Bus::fake()`, `Http::fake()` for their respective primitives. These are **not mocks** — they are in-memory fakes and are preferred over manual mocks.
- For adapters (third-party wrappers in `app/Adapters/`), hand-write a test fake class rather than using Mockery. Fakes are reused across tests; mocks duplicate assumptions.

## Forbidden

- `expect(true)->toBe(true)` and equivalent always-pass assertions.
- `it()->skip(...)` in main branches without a linked follow-up work item.
- Commented-out tests — delete them; git has memory.
- Time/date assertions that are not frozen (`Carbon::setTestNow(...)`).

## Coverage Target

- Unit: high coverage is cheap — aim for near-complete on pure logic.
- Feature: coverage is a lagging indicator. Every **acceptance criterion** in the Design Doc must map to at least one Feature test.

## Running Subsets

- `composer test -- --filter "it rejects"` for a name match.
- `composer test -- tests/Feature/SomeTest.php` for a single file.
- During development, use `--parallel` only when the test suite is known to be parallel-safe (no shared external state, no static singletons outside the container).

## Response JSON

Prefer `assertExactJson` only when the contract is small and stable. For most endpoints, combine `assertStatus` + `assertJsonPath`/`assertJsonStructure` so a new response field doesn't silently break every test.

## Comments in Tests

Test names carry the intent. Do not re-explain what `it('rejects a payload without an idempotency key')` does in a comment. Do document **why** a specific fixture or arrangement exists if it encodes a past bug.
