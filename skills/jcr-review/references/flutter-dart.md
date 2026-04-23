# Flutter / Dart — Framework-Specific Patterns

## Why It Matters

Flutter and Dart have unique patterns around widget composition, state management, and asynchronous programming. Misusing these patterns leads to unnecessary rebuilds, memory leaks, hard-to-debug state issues, and runtime crashes that the type system alone cannot prevent.

## Principles

- Prefer `StatelessWidget` unless mutable state is genuinely needed.
- Keep `build()` methods focused on layout. Extract logic into methods or separate widgets.
- Use `const` constructors wherever possible to enable widget tree optimization.
- Never use `BuildContext` across async gaps.
- Always clean up resources in `dispose()`.

## Checklist

### Widget Design

- [ ] `StatefulWidget` used when `StatelessWidget` would suffice (no `setState` call, or state could be lifted)
- [ ] `build()` method exceeding ~80 lines or containing business logic (calculations, formatting, network calls)
- [ ] Missing `const` constructor on widgets/classes with only final fields
- [ ] Missing `const` keyword on widget constructors in the widget tree (missed rebuild optimization)
- [ ] Deeply nested widget tree (3+ levels of nesting in a single `build()` method) — extract sub-widgets
- [ ] Missing `Key` parameter on widgets in `ListView`, `Column`/`Row` with dynamic children, or `AnimatedSwitcher`
- [ ] Creating widgets via helper methods (e.g., `Widget _buildHeader()`) instead of extracting into separate widget classes — prevents framework-level optimization

### State Management

- [ ] Calling `setState()` after `dispose()` (missing `mounted` check before `setState` in async callbacks)
- [ ] `setState()` wrapping logic that doesn't affect UI (e.g., updating a variable not used in `build()`)
- [ ] Performing heavy computation or I/O inside `setState()` — compute first, then call `setState` with the result
- [ ] Monolithic state: a single large `StatefulWidget` / provider managing unrelated pieces of state, causing unnecessary rebuilds across the tree
- [ ] Missing `dispose()` for `TextEditingController`, `ScrollController`, `AnimationController`, `FocusNode`, `StreamSubscription`, `Timer`

### Async / Error Handling

- [ ] Using `BuildContext` after an `await` without checking `mounted` (may reference a disposed element)
- [ ] Uncaught `Future` errors — fire-and-forget `Future` without `.catchError` or `try/catch`
- [ ] Missing error handling on `Stream.listen()` (no `onError` callback)
- [ ] `async` function whose return value is silently discarded (missing `unawaited()` or `await`)
- [ ] Using `FutureBuilder` / `StreamBuilder` without handling `ConnectionState.waiting` and `snapshot.hasError`

### Dart Conventions

- [ ] File names not in `snake_case` (Dart convention: `my_widget.dart`, not `MyWidget.dart` or `myWidget.dart`)
- [ ] Private members not prefixed with `_` (Dart uses `_` for library-level privacy)
- [ ] Using `dynamic` where a concrete type or generic could be specified
- [ ] Returning `null` from a function with a non-nullable return type annotation (or missing null-safety usage)
- [ ] Using `as` for type casting without prior `is` check — prefer pattern matching or `is` guard to avoid runtime exceptions
- [ ] Using `print()` for logging instead of `dart:developer`'s `log()` or a logging package
- [ ] Not using collection-if / collection-for / spread syntax where it simplifies widget trees or list construction

### Performance

- [ ] Rebuilding expensive widgets on every frame (missing `const`, `RepaintBoundary`, or `AutomaticKeepAliveClientMixin`)
- [ ] Allocating objects (lists, maps, closures) inside `build()` that could be hoisted to fields or `const`
- [ ] Using `ListView` instead of `ListView.builder` for large/dynamic lists
- [ ] Image assets not cached or improperly sized (missing `cacheWidth`/`cacheHeight`)
- [ ] Heavy synchronous work on the main isolate (JSON parsing, image processing) — should use `compute()` or isolates

## Good Examples / Bad Examples

Bad example — StatefulWidget without mutable state:
```dart
class Greeting extends StatefulWidget {
  final String name;
  const Greeting({required this.name});

  @override
  State<Greeting> createState() => _GreetingState();
}

class _GreetingState extends State<Greeting> {
  @override
  Widget build(BuildContext context) {
    return Text('Hello, ${widget.name}');
  }
}
```

Good example — StatelessWidget:
```dart
class Greeting extends StatelessWidget {
  final String name;
  const Greeting({super.key, required this.name});

  @override
  Widget build(BuildContext context) => Text('Hello, $name');
}
```

Bad example — BuildContext after await:
```dart
Future<void> _submit() async {
  final result = await api.submit(data);
  Navigator.of(context).pushReplacementNamed('/success');
}
```

Good example — mounted check:
```dart
Future<void> _submit() async {
  final result = await api.submit(data);
  if (!mounted) return;
  Navigator.of(context).pushReplacementNamed('/success');
}
```

Bad example — missing dispose:
```dart
class SearchPage extends StatefulWidget { ... }

class _SearchPageState extends State<SearchPage> {
  final _controller = TextEditingController();
  final _focusNode = FocusNode();

  // No dispose() — memory leak
}
```

Good example — proper cleanup:
```dart
class _SearchPageState extends State<SearchPage> {
  final _controller = TextEditingController();
  final _focusNode = FocusNode();

  @override
  void dispose() {
    _controller.dispose();
    _focusNode.dispose();
    super.dispose();
  }
}
```

Bad example — helper method returning widgets:
```dart
class ProfilePage extends StatelessWidget {
  Widget _buildAvatar(String url) {
    return CircleAvatar(backgroundImage: NetworkImage(url));
  }

  Widget _buildName(String name) {
    return Text(name, style: const TextStyle(fontSize: 24));
  }

  @override
  Widget build(BuildContext context) {
    return Column(children: [_buildAvatar(user.avatar), _buildName(user.name)]);
  }
}
```

Good example — extracted widget classes:
```dart
class ProfileAvatar extends StatelessWidget {
  final String url;
  const ProfileAvatar({super.key, required this.url});

  @override
  Widget build(BuildContext context) {
    return CircleAvatar(backgroundImage: NetworkImage(url));
  }
}
```

## Anti-patterns

- **God Widget** — A single widget with a 500-line `build()` method. Split into composable sub-widgets.
- **setState Overuse** — Using `setState` for everything instead of adopting a state management solution for cross-widget state.
- **BuildContext Smuggling** — Storing `BuildContext` in a field or passing it to a long-lived object. Context is tied to an element's lifecycle.
- **Stringly-Typed Routes** — Using raw strings for navigation (`'/user/detail'`). Prefer named routes, go_router, or typed route constants.
- **Uncontrolled FutureBuilder** — Placing a `FutureBuilder` whose `future` is created inside `build()`, causing the future to re-fire on every rebuild. Assign the `Future` to a field in `initState()`.
- **Catch-All Dynamic** — Using `dynamic` or `Map<String, dynamic>` for everything to avoid writing model classes. Loses type safety and IDE support.
- **Print Debugging in Production** — Leaving `print()` calls in production code. Use structured logging or `dart:developer` `log()`.
