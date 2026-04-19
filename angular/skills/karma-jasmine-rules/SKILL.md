---
name: karma-jasmine-rules
description: Karma + Jasmine testing rules for Angular 18 projects. Use when writing or modifying tests in an Angular codebase configured with Karma and Jasmine.
---

# Karma + Jasmine Testing Rules (Angular 18)

## Runner

- `ng test` (canonical). `npm test` maps to the same.
- Karma config: `karma.conf.js` at the frontend root. Do not modify without an ADR — the CI pipeline depends on its shape.
- Browsers: headless Chrome in CI, `--browsers Chrome` locally for debugging.

## Organisation

- Test file co-located with the unit: `foo.component.ts` ↔ `foo.component.spec.ts`.
- Service tests: `foo.service.spec.ts`.
- One `describe` per unit; nest `describe` by scenario only when it adds readability.

## Style

- Jasmine `describe` / `it` / `expect`. Prefer `it('should ...')` phrasing — the Angular community default.
- Arrange-Act-Assert, with blank lines between sections in tests over ~6 lines.
- One behaviour per `it`. Resist bundling unrelated assertions.

## Component Tests

- Use Angular `TestBed` with standalone components via `TestBed.configureTestingModule({ imports: [FooComponent] })`.
- **Interact through the DOM**: `fixture.debugElement.query(By.css('[data-test-id="submit"]'))`. Use `data-test-id` attributes as stable hooks; do not query by CSS class or structural selector.
- Trigger change detection explicitly via `fixture.detectChanges()`. For signal-driven components, `TestBed.flushEffects()` or `fixture.autoDetectChanges(true)` when effects are involved.
- For Material / PrimeNG overlays, use harnesses where available (`@angular/cdk/testing`, `MatInputHarness`, etc.) rather than DOM queries.

## Service Tests

- No `TestBed` needed for pure logic — instantiate directly with `new FooService(deps...)`.
- For services that inject `HttpClient`, use `HttpTestingController` + `provideHttpClientTesting()`. Verify `httpMock.verify()` in `afterEach`.
- For services consuming signals from other services, pass a fake that returns signal values — don't reach into private state.

## Mocking

- **Prefer fakes to spies**. A hand-written class that implements the interface survives refactors; a spy dies the moment the method name changes.
- `jasmine.createSpy` for single-function callbacks only.
- Never mock Angular primitives (HttpClient, Router) with spies — use `provideRouter(...)` for real routing, `HttpTestingController` for HTTP.

## RxJS in Tests

- Cold observables with `marbles` when behaviour depends on timing (rare).
- `firstValueFrom` + `async/await` when the stream is expected to emit once.
- `fakeAsync` + `tick()` for timer-dependent code. Do not mix `fakeAsync` with Promises unless you explicitly `flushMicrotasks()`.

## Signals in Tests

- Read signal values with the signal function call: `expect(sut.count()).toBe(3)`.
- Trigger changes by setting input signals: `fixture.componentRef.setInput('userId', 'u1')` then `fixture.detectChanges()`.
- Effects: run them via `TestBed.flushEffects()` (Angular 18+).

## Forms

- For reactive forms, drive the test by patching the `FormGroup`, not the DOM — except for tests whose purpose is template wiring.
- Validator tests are pure functions: `expect(customValidator({ value: '' }))` — no `TestBed`.

## What to Test

| Layer | What to test | Where |
|---|---|---|
| Component | Rendering given inputs, output events on user interaction, template error state | `.component.spec.ts` |
| Service | Business logic, HTTP flows (with `HttpTestingController`), signal/observable contracts | `.service.spec.ts` |
| Guard / Resolver | Allow/deny logic for each branch, resolved data shape | `.guard.spec.ts` / `.resolver.spec.ts` |
| Pipe | Pure input-output for every branch | `.pipe.spec.ts` |
| Directive | DOM side effects applied/removed | `.directive.spec.ts` |

## What NOT to Test

- Angular framework itself (router navigation happened, lifecycle hooks fired — Angular tests those).
- Material / PrimeNG internals.
- Styles / visual layout — out of scope for unit tests; belongs in visual regression or Storybook.

## Async Assertions

- `fakeAsync` + `tick()` for deterministic timer control.
- `await fixture.whenStable()` after async-initiated change detection.
- Avoid `setTimeout` in tests — always use `tick()`.

## Forbidden

- `xit` / `xdescribe` in main branches without a linked follow-up.
- `expect(true).toBeTrue()` and equivalent tautologies.
- Deleting/rewriting a failing test to match new behaviour without first confirming the behaviour change is intentional.
- `spyOn(console, 'error')` to silence warnings — fix the underlying cause.

## Coverage

- `ng test --code-coverage` outputs to `coverage/`. Aim for high coverage on services and pipes; component coverage is secondary — behaviour coverage matters more than line coverage.
- Zero new uncovered branches for shipped business logic.

## Performance

- Tests run in parallel per test file in CI. Don't rely on shared mutable state between files.
- Clean up subscriptions with `takeUntilDestroyed` or explicit teardown — leaked subscriptions cause flake.

## Filters

- `ng test --include '**/foo.component.spec.ts'` for a single file.
- `fdescribe` / `fit` for focus during local development — **must not** land in a commit. Lint the codebase for these.

## Comments in Tests

Test names are the documentation. Add a comment only to record a gotcha that future-you will otherwise relearn (e.g. "Material's Overlay renders outside the component, query `document.body`").
