# Layout And Module Placement

## Canonical Layout

The canonical directory tree is defined in the **Canonical Layout** section of
[`SKILL.md`](../SKILL.md). Treat that tree as the single source of truth, place
every file in its canonical owner under `src/`, and organize directories by
responsibility, not by team preference or historical drift.

The app root is `src/` (the Rayfin template convention, aliased as `@/*`), not
`app/`. There is no server layer: the backend is the Rayfin/Fabric platform,
reached only through `src/lib/infrastructure/`.

## Mandatory Placement Preflight

Before implementing, list every file you expect to create, move, or extract and
assign its exact target path.

Use this rule:

1. every file must have one canonical owner before code is written
2. if a file has no clear owner, revise the placement plan first
3. do not create a new directory just because the first idea does not fit the
   layout

Treat convenience directories such as `src/features/`, `src/modules/`,
`src/hooks/`, `src/services/`, `src/utils/`, `src/types/`, or `src/store/` as
layout drift unless an explicit migration plan says otherwise.

## Mapping The Rayfin Template Onto The Layers

The `npm create @microsoft/rayfin` template ships a flat starter structure.
Map it onto the clean layers as you grow the app:

- template `src/services/rayfinClient.ts` → `src/lib/infrastructure/rayfin/`
- template `src/services/*AuthService.ts` (+ `IAuthService`) →
  auth port in `src/lib/domain/ports/` + implementations in
  `src/lib/infrastructure/auth/`
- template `src/services/todos.ts` → repository port in
  `src/lib/domain/repositories/` + implementation in
  `src/lib/infrastructure/data/`
- template `src/services/bootstrap.ts` → composition root in `src/main.tsx`
  plus factories in `src/lib/infrastructure/config/`
- template `src/hooks/AuthContext.tsx` → `src/lib/usecase/auth/`
- template `src/pages/`, `src/components/` → unchanged in spirit, kept thin

The platform files under `rayfin/` (`data/*.ts` entities, `rayfin.yml`) stay
where they are and are owned by the `rayfin` skill, not this one.

## Feature Presentational Component Placement

Keep feature-specific presentational components in `src/components/<feature>/`.

Preferred shape:

```text
src/components/todo/
  TodoList.tsx
  TodoListItem.tsx
  TodoComposer.tsx
```

One component per `.tsx` file, and the file name matches the exported
component name. Style with Tailwind utility classes. See
[`component-and-styling-rules.md`](component-and-styling-rules.md) for the full
set of component and styling rules.

Use `src/components/shared/` only when the component is feature-agnostic,
reusable across features, and expressed in generic UI language instead of
product vocabulary.

## State Module Placement

For non-trivial client interaction flows, create a feature directory under
`src/lib/usecase/` and colocate the state modules there.

Preferred shape:

```text
src/lib/usecase/todo/
  use-todo.ts
  state.ts
  reducer.ts
  selectors.ts
  handlers.ts
  types.ts
```

Use these files by responsibility:

- `state.ts`: state shape and initial state
- `reducer.ts`: reducer function and action definitions
- `selectors.ts`: derived read models and computed flags
- `handlers.ts`: event-to-dispatch mapping or async command helpers (calling
  repository ports through the use case)
- `use-<feature>.ts`: public Hook or controller entry point

Do not create horizontal buckets such as `src/state/`, `src/reducers/`,
`src/stores/`, `src/handlers/`, or `src/lib/usecase/state/`.

## Domain And Infrastructure Placement

- Ports (interfaces the app depends on) live in `src/lib/domain/`:
  persistence ports in `repositories/`, other outbound ports (auth, clock,
  notifier) in `ports/`.
- Adapters (concrete implementations) live in `src/lib/infrastructure/`:
  the RayfinClient facade in `rayfin/`, repository implementations in `data/`,
  auth-service implementations in `auth/`, browser adapters in `browser/`, env
  readers and factories in `config/`.
- Domain models, value objects, and policies live in `src/lib/domain/` and
  never import React, the Rayfin SDK, browser APIs, or the `rayfin/data`
  decorator entities.

## No Generic Common Bucket

Do not add a catch-all common directory.

Prefer this order instead:

1. Keep code with the use case, domain, or infrastructure module that owns it.
2. Duplicate a tiny utility once if the reuse pattern is still uncertain.
3. Extract only after the abstraction is clearly stable.

## Feature Public API Rule

Treat each feature directory as having a public entry point and private
internals.

Prefer:

- `lib/usecase/<feature>/use-<feature>.ts` as the public use-case entry
- one primary port or adapter module as the public data entry

Avoid importing another feature's private files such as `reducer.ts`,
`selectors.ts`, `handlers.ts`, or `state.ts`.

## Circular Dependency Rule

Treat import cycles as architecture violations, especially inside feature
internals.

Common bad cycles:

- `selectors -> handlers -> reducer -> selectors`
- `page -> usecase -> page`
- `infrastructure -> usecase -> infrastructure`

When a cycle appears:

1. move shared pure logic to a lower-level leaf module
2. invert the dependency through a port or parameter
3. merge modules back together if the split was artificial

## TS And TSX Naming Rule

Keep file names explicit enough that responsibility is visible before opening
the file.

Use these defaults:

- page containers: `PascalCase.tsx` ending in `Page`, such as `TodoPage.tsx`
- React component files: `PascalCase.tsx`
- non-component modules: responsibility-based `kebab-case.ts`, such as
  `todo-repository.ts`

For component files:

- one React component per `.tsx` file
- the primary exported component name matches the file name exactly
  (`TodoList.tsx` exports `TodoList`)
- use `.tsx` only when the file renders JSX
- keep feature-specific views under `src/components/<feature>/` until they
  prove generic enough for `src/components/shared/`

See [`component-and-styling-rules.md`](component-and-styling-rules.md) for the
full component and Tailwind conventions.

Inside a feature directory, short file names such as `state.ts`, `reducer.ts`,
`selectors.ts`, `handlers.ts`, and `types.ts` are acceptable because the
directory already provides the feature context.

Avoid vague file names such as `helpers.ts`, `utils.ts`, `common.ts`,
`misc.ts`, `temp.ts`, or `new.ts`.

Use `index.ts` only when it is a deliberate public entry point.

## Typical Flow

For creating a todo:

1. `App.tsx` mounts a route to a thin `src/pages/TodoPage.tsx`.
2. A client use-case Hook (`src/lib/usecase/todo/use-todo.ts`) owns the draft
   state and handlers.
3. A presentational component (`src/components/todo/TodoComposer.tsx`) renders
   fields and calls passed handlers.
4. The handler calls a repository port through the use case.
5. The repository implementation (`src/lib/infrastructure/data/todo-repository.ts`)
   runs `client.data.Todo.create(...)` and maps the result to a view model.

Each step should only depend on the next layer inward or on a port explicitly
created for that boundary.
