---
name: angular-rules
description: Angular 18 + TypeScript + Material/PrimeNG rules for frontend development. Use when implementing Angular components, services, state management, routing, or forms.
---

# Angular Development Rules

## Baseline

- **Angular**: 18.2.x. Use standalone components by default; NgModules only when integrating with a legacy area that still requires them.
- **TypeScript**: strict mode. `noImplicitAny`, `strictNullChecks`, `strictTemplates` in `angularCompilerOptions`.
- **UI libraries** (in priority order when both offer the same primitive): **Angular Material** first; **PrimeNG** when Material doesn't cover the need (rich data grids, specialised inputs, complex overlay patterns).
- **RxJS**: 7.8. Use operators; avoid subscribing in templates manually — use `async` pipe or `toSignal`.
- **Signals**: preferred for local component state and for derived state. Keep RxJS for async streams (HTTP, WebSocket, router events, form value changes).
- **Style**: SCSS. Customised Material theme at `src/styles`.

## Type Safety

- Replace every `any` with `unknown` + type guards, or a precise interface/type.
- `as` casts only at trusted boundaries (HTTP response parsing after validation, router data with typed resolvers).
- `strictTemplates: true` — do not disable; instead tighten the model.

## Component Design

- **Standalone components** (`standalone: true`) with explicit `imports: [...]`. Do not re-export components through intermediate modules.
- **ChangeDetection.OnPush** by default. `Default` is allowed only when Material/PrimeNG components force it.
- **Inputs/Outputs** declared with `input()` / `output()` signals (Angular 17.1+ API). Required inputs use `input.required<T>()`.
- **Computed state** via `computed(...)`; `effect(...)` only for side effects that genuinely need to fire on signal change.
- **Component size**: 300 lines is a smell; split presentational from container when data-fetching appears alongside complex layout.
- **No logic in templates** beyond signal reads, pipes, and `@if` / `@for` control flow. Control flow uses the new `@if`/`@for`/`@switch` blocks (Angular 17+), not the `*ngIf` / `*ngFor` structural directives for new code.

## State Management

- **Local state**: signals (`signal()`, `computed()`, `effect()`).
- **Shared state across features**: service with signals + `BehaviorSubject` for async sources. Inject with `inject()` in the service constructor site; prefer `providedIn: 'root'` unless the service is feature-scoped.
- **Server state**: RxJS + `HttpClient`; cache with `shareReplay({ bufferSize: 1, refCount: true })` when appropriate. For optimistic updates, keep the source of truth in a signal and reconcile on response.
- **No NgRx** in this codebase — do not introduce it without an ADR.

## Routing

- Standalone route configs (`provideRouter(...)`, `Routes`). Lazy-load feature areas with `loadChildren: () => import(...)`.
- Guards with functional syntax (`CanActivateFn`). Inject dependencies via `inject()`.
- Resolvers return typed data; the component consumes via `withComponentInputBinding()`.

## Forms

- **Reactive forms** default; template-driven forms only for trivial single-input cases.
- Typed `FormGroup<...>` everywhere — `FormGroup<{ email: FormControl<string>; ... }>`.
- Validators from `@angular/forms`; custom validators are pure functions returning `ValidationErrors | null`.

## HTTP

- Responses typed, validated on entry. Adapt API shape to UI shape in a typed mapper function — components consume domain types, not API types.
- Interceptors for auth, tracing, error normalisation. Keep them focused — one concern per interceptor.

## Material + PrimeNG

- **Do not mix** two components with the same role on the same screen (e.g. `mat-table` + `p-table`). Pick one per feature.
- Custom theming goes through Material's design tokens for Material parts and PrimeNG's CSS variables for PrimeNG parts — no manual colour values in component styles.
- Icons: `ng-icons` wrappers; do not import raw Material icon font alongside PrimeIcons unless both are actively used.

## Asynchronous Handling

- `async/await` on promises; RxJS operators on observables.
- Every pipeline has a terminal operator or an explicit unsubscribe — prefer `takeUntilDestroyed()` over manual `ngOnDestroy` + `Subject`.
- `firstValueFrom` / `lastValueFrom` when promise semantics are appropriate; never mix `.subscribe(...)` with awaiting a promise for the same stream.

## Errors

- HTTP errors normalised in an interceptor to a typed `AppError` union (`ValidationError | AuthError | NotFoundError | ServerError`).
- Components surface user-facing errors via a shared `ErrorDisplay` primitive, not ad-hoc `console.error`.
- Unhandled errors from signals / effects hit a global `ErrorHandler` — register one in `provideErrorHandler`.

## Performance

- `OnPush` everywhere. Lists render via `@for` with a `track` expression referencing a stable id.
- Heavy work off the render path: Web Workers (`worker-plugin`) for serialization-heavy computation, `requestIdleCallback` for deferred enrichment.
- Bundle watched via `ng build --configuration production --stats-json` + `source-map-explorer`. Pre-existing budgets in `angular.json` must not regress.

## Security

- All secrets stay backend-side. Env values in `src/environments/*.ts` are public — treat them as config, not secrets.
- Do not use `[innerHTML]` with user input unless piped through `DomSanitizer.bypassSecurityTrustHtml` AND the input is demonstrably safe.
- CSP: the app is served behind a CSP — do not introduce inline scripts/styles.

## Build Target Quirk

The Angular production build outputs to `../editor/web` (per `angular.json`). Do not change this path — the Flutter editor embeds the build. Any task that touches `angular.json` must preserve `outputPath`.

## Format

- Prettier + Angular ESLint. Import order grouped by: Angular → RxJS → third-party → internal absolute → relative.
- Kebab-case filenames (`user-profile.component.ts`); PascalCase types; camelCase variables.

## Comments

Default to none. Add a short comment only for non-obvious "why".

## Clean Code

- Delete unused imports, components, exports, dead `@ViewChild` refs immediately.
- No leftover `console.log` / `debugger` / `ng.probe` calls.
