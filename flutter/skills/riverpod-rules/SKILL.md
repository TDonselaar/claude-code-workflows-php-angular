---
name: riverpod-rules
description: Riverpod + riverpod_annotation (code-gen) state management rules for Flutter. Use when implementing providers, notifiers, or consuming state in widgets.
---

# Riverpod Rules (flutter_riverpod + riverpod_annotation)

## Baseline

- Package: `flutter_riverpod` + `riverpod_annotation` (code-gen). Annotation-driven providers are the default; legacy `StateProvider`/`Provider` constructors allowed only in files that haven't been migrated yet
- Generated files: `*.g.dart` committed. Regenerate with `fvm dart run build_runner build --delete-conflicting-outputs`
- Root: `ProviderScope` wraps the app at entry point (`main.dart`). Test setups use a fresh `ProviderContainer` or `ProviderScope(overrides: [...])`

## Provider Choice

Decision ladder:

1. **Stateless, derived value** (sync function of other providers) → `@riverpod` function returning a value or a `FutureOr<T>`
2. **State you mutate** → `@riverpod class Foo extends _$Foo` with methods
3. **Stream source** (WebSocket, polling, Firebase listener) → `@riverpod` function returning `Stream<T>` or a notifier with `build` returning `Stream<T>`
4. **External sync value** (from platform channel, package that exposes a callback API) → wrap in a notifier

Legacy API → code-gen mapping:

| Legacy | Code-gen |
|---|---|
| `Provider<T>` | `@riverpod T foo(Ref ref)` |
| `FutureProvider<T>` | `@riverpod Future<T> foo(Ref ref) async` |
| `StreamProvider<T>` | `@riverpod Stream<T> foo(Ref ref)` |
| `StateNotifierProvider` | `@riverpod class Foo extends _$Foo` |
| `StateProvider` | Same as StateNotifier — `StateProvider` is discouraged |

## Lifecycle

- **`autoDispose`** by default — `@Riverpod(keepAlive: false)` (the default) OR annotation without `keepAlive: true`. Keep-alive only for genuinely global caches (current user, auth token, feature flags)
- **`ref.onDispose`** — register cleanup (stream subscriptions, timers, platform listeners) inside the provider body. If a resource isn't disposed, it's a leak
- **`ref.keepAlive()`** — use sparingly inside autoDispose providers to extend lifetime after a one-shot operation (e.g. cache result for 30s); always cancel the keepAlive link via `link.close()` on a timer

## Arguments & Family

- **Family providers** via annotated function parameters: `@riverpod User user(Ref ref, String userId)` generates a family keyed by `userId`
- Arguments must be **value-equatable** — Riverpod uses `==` to dedupe. Use primitive types, enums, or classes with generated `==`/`hashCode`. Avoid passing raw `Map`/`List` — wrap in a typed model

## Mutation Pattern

Notifier pattern:

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    final repo = ref.watch(todoRepoProvider);
    return repo.fetchAll();
  }

  Future<void> add(String title) async {
    final repo = ref.read(todoRepoProvider);
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      await repo.add(Todo(title: title));
      return repo.fetchAll();
    });
  }
}
```

Rules:

- `build()` is the source of truth — it establishes initial state. Other methods mutate via `state = ...`
- Wrap mutations in `AsyncValue.guard` when the result is `AsyncValue<T>` — it converts exceptions into `AsyncError` states uniformly
- Do not call `ref.invalidate(self)` from inside a notifier method — let callers invalidate if they need a fresh build

## Reading State in Widgets

- `ref.watch(provider)` in `build()` — causes rebuilds
- `ref.read(provider.notifier)` — for one-shot method calls (button press, form submit). Never `ref.read` a value-returning provider in `build`
- `ref.listen(provider, (prev, next) => ...)` — for side effects (navigation, SnackBar, analytics) that depend on state changes. In `build()` for `ConsumerWidget`, in `build()` or `initState` for `ConsumerStatefulWidget`
- **Selector pattern**: `ref.watch(provider.select((s) => s.field))` when only one field matters — avoids unnecessary rebuilds

## AsyncValue Handling

- Always handle all three branches: `data`, `loading`, `error`. Use `when(data:, loading:, error:)` or `maybeWhen`/`whenOrNull` with explicit fallback
- Skeleton loaders for first-load; inline spinners for refresh. Riverpod distinguishes `isLoading` vs `isRefreshing` (`asyncValue.isLoading && !asyncValue.isRefreshing`)
- Error UI is a first-class concern, not a fallback. Map `AsyncError` to typed failures in the provider layer; widgets present user-friendly messages

## Dependencies Between Providers

- `ref.watch(otherProvider)` — rebuild when `otherProvider` changes
- `ref.read(otherProvider)` — one-shot dependency, no rebuild
- Do not introduce circular dependencies. If A depends on B and B depends on A, refactor — usually one of them should consume an event stream instead of watching the other

## Testing Providers

- Create a fresh `ProviderContainer` per test. Dispose in `tearDown`
- Override external dependencies with `ProviderScope(overrides: [...])` or `container.read` against a test-only provider
- For notifier tests, instantiate via `container.read(fooProvider.notifier)` and assert on `container.read(fooProvider)` transitions
- For widget tests, wrap `pumpWidget` in `ProviderScope(overrides: [...])` and use `find.byKey` or `find.byType` with `tester.state` for notifier access if needed

## Forbidden

- Storing a `BuildContext` in a provider (leak + decoupling violation)
- Mutating a widget's `State` from inside a provider — widgets listen, providers don't reach out
- `GlobalKey`s as identifiers for providers — use family with typed IDs
- Reading `ref` after dispose — always check via lifecycle guards when scheduling async callbacks

## Code-gen Conventions

- One feature = one file (e.g. `todos.dart` with `todos.g.dart`); larger features split by concern
- Keep annotation arguments minimal — `@riverpod` alone is the default; `@Riverpod(keepAlive: true)` only when justified in a comment

## Migration (legacy → code-gen)

When touching a legacy provider in an active task:

- Introduce the annotated version alongside
- Migrate all call sites in the same task
- Remove the legacy provider at the end — don't ship both permanently

If the call-site count is large (say > 20), split into a dedicated migration task with its own task file and ADR entry.
