# View State And Handler Patterns

## When To Add A View State Module

Add a use-case module when at least one is true:

- the view has more than one piece of related state
- multiple handlers mutate that state
- async fetching, optimistic updates, or retries are involved
- derived values are non-trivial
- the same screen needs to be reused or tested with different data

For a single boolean toggle, keep state in the component.

## Suggested Structure

For a feature `todo`:

```text
src/usecase/todo/
  use-todo.ts
  state.ts
  reducer.ts
  selectors.ts
  handlers.ts
  types.ts
```

Use this responsibility split:

- `types.ts`: state-only types used by reducer, selectors, and handlers
- `state.ts`: initial state factory and state shape definition
- `reducer.ts`: reducer plus intent-named action types
- `selectors.ts`: derived values such as visible items, computed flags, view
  models (the Presenter output)
- `handlers.ts`: event-to-dispatch mapping and async command helpers that call
  repository ports through the use case
- `use-todo.ts`: the public Hook that receives ports and exposes the view model

For very small features, two files such as `use-feature.ts` and `state.ts`
are enough.

## Data Access From A Use Case

- The use case receives repository ports (and other ports) by injection —
  through its Hook argument list or a React Context supplied by the composition
  root.
- Handlers call those ports; they never import `client.data`, `RayfinClient`,
  or an auth SDK.
- Map port results into view state; expose a view model, not raw rows.

```ts
export function useTodo(deps: { todos: TodoRepository } = useDeps()) {
  const [state, dispatch] = useReducer(reducer, undefined, initialState);
  const view = selectTodoView(state);

  const createTodo = useCallback(async (title: string) => {
    dispatch({ type: "todoCreateRequested" });
    try {
      const todo = await deps.todos.create({ title });
      dispatch({ type: "todoCreated", todo });
    } catch (err) {
      dispatch({ type: "todoCreateFailed", error: toMessage(err) });
    }
  }, [deps.todos]);

  return { view, createTodo /* … */ };
}
```

## Component Contract

Components should:

- accept props from the use case (via the page container)
- call passed handlers
- render JSX styled with Tailwind
- hold ephemeral UI state only when it is genuinely view-local

Components should not:

- import `client.data`, `RayfinClient`, or repository modules
- call ports or the Rayfin client directly
- own reducer logic or async orchestration
- own derived business view models

## Component File And Styling Rules

Apply these in addition to the contract above:

- One React component per `.tsx` file. The file name (`PascalCase.tsx`) and the
  primary exported component name match exactly.
- Co-define the component's `Props` type in the same file.
- Style with Tailwind utility classes; do not use CSS Modules or inline `style`
  for static styling.

See [`component-and-styling-rules.md`](component-and-styling-rules.md) for the
full set of file-shape and Tailwind conventions.

## Action Granularity

Prefer descriptive action names that match user intent:

- `todoCreateRequested`
- `todoCreated`
- `completionToggled`
- `todoCreateFailed`

Avoid generic action names such as `SET_STATE`, `UPDATE_FIELD`, or `RESET` when
they hide what the user actually did.

## Selector Rule

Keep selectors pure and dependency-free.

Selectors should:

- take the state shape as input
- return derived values / the view model
- not call ports or the Rayfin client
- not trigger side effects
- not mutate state

When a derived value needs freshly fetched data, do the fetch in the use-case
Hook, not in a selector.

## Handler Rule

Handlers can:

- read current state through a passed selector or snapshot
- call async dependencies through injected ports
- dispatch reducer actions

Handlers should not:

- own UI rendering
- import React components
- write directly to module-level mutable state

## Hook Contract

A `use-<feature>.ts` Hook should:

- receive its dependencies (ports) through its argument list or React Context
- expose a small interface (a view model plus handlers) to the view layer
- combine state, selectors, and handlers into a single object
- own subscription cleanup

A view should be able to swap one Hook implementation, or one injected port
implementation, for another in tests without changing the component tree. This
is what makes the Strategy and Ports-and-Adapters patterns pay off.
