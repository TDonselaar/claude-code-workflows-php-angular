---
name: flutter-rules
description: Flutter + Dart rules for multi-platform apps (Android, iOS, desktop, web, Tizen). Use when implementing Flutter widgets, routing, platform channels, services, or build configuration.
---

# Flutter Development Rules

## Baseline

- **Dart SDK**: per `pubspec.yaml` (edge channel in this codebase, `>=3.7.0-0 <4.0.0`)
- **Flutter version**: pinned via **FVM** (`.fvmrc`). Always invoke Flutter through `fvm flutter ...` in scripts and CI — never the system `flutter`
- **Platforms**: Android, iOS, Linux, macOS, Windows, Web (embeds Angular build at `build/web`), Samsung Tizen. Platform-specific code lives in `lib/platform/<name>/` or behind `kIsWeb` / `Platform.is*` guards
- **Null safety**: sound null safety everywhere. No `!` unwrap without a nearby reason comment

## Directory Layout

- `lib/main.dart` — entry per platform target
- `lib/screens/` — screen-level widgets (routed destinations)
- `lib/widgets/` — reusable widgets
- `lib/services/` — domain services (HTTP, storage, platform integration)
- `lib/models/` — data models; immutable by default (`final`, no setters, `copyWith`)
- `lib/layer_types/` — domain-specific type definitions
- `lib/js/` — web/JS interop
- `lib/tools/` — dev tooling / generators
- `lib/providers/` (or `lib/state/`) — Riverpod providers
- `test/` — unit + widget tests (mirror `lib/` structure)

## Widget Design

- **Prefer composition over configuration**. Small widgets with clear intent beat giant configurable widgets
- **`const` constructors** wherever possible. A non-`const` widget that could be `const` is a perf bug
- **`StatelessWidget`** by default. `ConsumerWidget` / `ConsumerStatefulWidget` (Riverpod) when reading providers. Raw `StatefulWidget` only when lifecycle hooks are genuinely needed and can't be expressed via providers
- **`build()` must be pure** — no side effects, no HTTP, no navigation. Side effects go in callbacks or `initState` / Riverpod `ref.listen`
- **Keys** — add them only when they matter (list reconciliation with stable identity, preserving state across tree changes). Do not sprinkle `Key` parameters decoratively
- **Breakpoint responsive layout** — `LayoutBuilder` + `MediaQuery` to adapt, not platform branching

## Routing

- `go_router` preferred (if already in `pubspec.yaml`); otherwise `Navigator 2.0` declarative API with typed route classes — not string-based `Navigator.pushNamed`
- Deep links defined with typed parameters. Every route has a redirect rule for missing auth
- Route guards live alongside the route config, not inside screen widgets

## Platform Channels

- Channel names namespaced: `com.example.app.<feature>`
- Type-safe wrappers in `lib/services/platform/` — widgets consume the wrapper, never `MethodChannel` directly
- Errors from the platform side surface as typed Dart exceptions, not raw `PlatformException`

## State

See `riverpod-rules` skill. Short version: Riverpod with code-gen annotations. No `setState` outside animation controllers or local-to-widget transient state that cannot be expressed with a provider.

## HTTP / IO

- `dio` or `http` — whichever the codebase already uses consistently. Do not introduce a second client
- Responses parsed into typed models; never pass `Map<String, dynamic>` across service boundaries
- Network errors mapped to typed failures (`NetworkFailure`, `AuthFailure`, `ServerFailure`) before surfacing to providers/widgets
- Timeouts explicit, never default
- For streaming (WebSocket / SSE), lifecycle tied to a Riverpod `StreamProvider` or `AutoDisposeStreamProvider`

## Async / Streams

- `async`/`await` for futures. Never use `.then(...)` chains — they hide exceptions
- Every `Future` returned from a public API is either awaited or explicitly `unawaited(...)` — no silent drops
- `Stream`s terminated with `takeUntil`-style patterns via Riverpod lifecycle, or manual `StreamSubscription.cancel()` in `dispose`

## Errors

- `Error` vs `Exception`: throw `Exception` subtypes for recoverable conditions; `Error` only for programming mistakes (e.g. `StateError` for invalid state that should never happen)
- Top-level errors caught by `runZonedGuarded` / `FlutterError.onError` → report to Firebase Crashlytics
- Never `catch (e) { }` silently. Either handle, rethrow, or `Crashlytics.recordError(e, stack)`

## Models

- Immutable (`final` fields, no setters)
- `copyWith` generated or hand-written; `==` / `hashCode` / `toString` generated via `freezed` or `equatable` (pick the one already in use)
- JSON codecs via `json_serializable` + code-gen; hand-written only for trivial cases

## Logging

- `logger` package (if present) or a thin `AppLog` wrapper. Structured fields, never concatenated strings
- Redact tokens, passwords, PII before logging
- No `print()` in shipped code — lint-enforced

## Theming

- Central `ThemeData` per brand/platform variant in `lib/theme/`
- Colours/typography via theme tokens — `Theme.of(context).colorScheme.primary`, not hardcoded `Color(0xff...)`
- Dark mode handled by `ThemeMode.system` + both `light` and `dark` `ThemeData`s

## Internationalization

- ARB files in `lib/l10n/`. Generated `AppLocalizations` via `flutter gen-l10n`
- Strings in widgets come from `AppLocalizations.of(context)!` — no hardcoded user-visible text

## Platform Differences

- `kIsWeb` for web branches
- `Platform.isAndroid` / `Platform.isIOS` etc. guarded behind `!kIsWeb` (the `Platform` API is unavailable on web)
- Tizen-specific code behind a build-time flag / separate entry point (`main_tizen.dart`)

## Build & Tooling

- `fvm flutter analyze` — zero issues, zero warnings
- `fvm flutter test` — all pass
- `fvm dart format --set-exit-if-changed .` — project formatted
- Code generation: `fvm dart run build_runner build --delete-conflicting-outputs` (when models/providers use code-gen) — committed outputs under `.g.dart` / `.freezed.dart`
- Analyzer config: `analysis_options.yaml` — do not relax rules without ADR

## Performance

- Avoid rebuilding large subtrees — use `const` widgets, `Selector`-like patterns in Riverpod (`ref.watch(provider.select((s) => s.field))`)
- `ListView.builder` (or `SliverList` with `SliverChildBuilderDelegate`) for lists > 20 items — never map a list to `Column` children for long lists
- Images: `cacheWidth` / `cacheHeight` when rendering resized thumbnails

## Security

- API keys / tokens in secure storage (`flutter_secure_storage`) on mobile, platform keystore on desktop, not `SharedPreferences`
- Firebase config files (`google-services.json`, `GoogleService-Info.plist`) via secret provisioning in CI, not committed
- Webview `javascriptMode` disabled unless explicitly needed; target URLs allow-listed
- Platform channel inputs validated on both sides

## Comments

Default to none. Add a short "why" when a widget contains a workaround (e.g. "Tizen clips overflow at the system overlay boundary — pad by 8dp"). Do not comment the what.

## Clean Code

- Remove unused widgets, unreferenced providers, leftover `_experimental` paths
- Delete commented-out widget trees — version control remembers
