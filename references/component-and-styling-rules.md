# Component File And Styling Rules

Use these rules whenever you create or modify a React component, its styling,
or its file layout. They sit alongside
[`layout-and-module-placement.md`](layout-and-module-placement.md) and
[`view-state-and-handler-patterns.md`](view-state-and-handler-patterns.md) and
take precedence on file-shape and styling questions.

Rayfin apps style with **Tailwind CSS** (the template ships Tailwind v4 via
`@tailwindcss/vite`). This skill does not use CSS Modules and does not require a
general-purpose component library.

## Goal

- Keep each component independently readable, movable, and testable.
- Keep styling consistent through Tailwind utilities and a shared set of
  presentational primitives.
- Keep view logic out of components (see the use-case patterns).

## One Component Per File

- A `.tsx` file exports exactly one React component as its primary, file-named
  export.
- File name and component name match exactly. `TodoList.tsx` exports
  `TodoList`; `TodoComposer.tsx` exports `TodoComposer`.
- Do not co-define a second top-level component in the same file just because
  the two are used together. Each gets its own file. (A page or `App.tsx` may
  keep one tiny private guard/helper component, but its primary export still
  matches the file name.)
- Tiny presentational helpers that are clearly private to the component may
  stay in the same file only when all of the following hold:
  - it is not exported
  - it does not hold its own state or effects
  - extracting it would not improve readability
  When in doubt, give it its own file.
- Pure utility helpers, hooks, types, and constants used by the component
  belong in sibling files, not appended to the component file.

## Filename And Export Conventions

- Component files: `PascalCase.tsx` matching the exported component name.
- Page containers: `PascalCase.tsx` ending in `Page` under `src/pages/`.
- Use `.tsx` only when the file renders JSX. Plain logic stays in `.ts`.
- Prefer named exports for components. A `default` export is acceptable for the
  root `App` and page modules if the project's convention uses it, but do not
  default-export shared primitives.
- Do not introduce barrel `index.ts` files inside `src/components/<feature>/`
  by default. Import components directly from their file.
- Co-define the component's prop type in the component file as
  `type <ComponentName>Props = { ... }`. Move it to a sibling
  `<ComponentName>.types.ts` only when shared with another module in the same
  feature.

## Component File Layout

Inside a component file, keep this order:

1. imports (React, sibling modules, then types)
2. local types (`type TodoListProps = { ... }`)
3. the component function declaration
4. small private sub-components or helpers, if any, that satisfy the
   one-file-helper exception above

Do not let a component file own:

- async orchestration → put it in `src/lib/usecase/<feature>/`
- reducer logic → same
- data access → a repository port called from the use case; never `client.data`
  or `RayfinClient` in a component

## Tailwind Styling Rules

- Style with Tailwind utility classes in `className`. Use the design scale
  (spacing, color, typography tokens) rather than arbitrary values; reach for
  `[...]` arbitrary values only for genuine one-offs.
- Keep class lists readable: group by concern (layout, spacing, color, state)
  and order consistently. Extract when a class list becomes long and repeated.
- Keep repeated utility clusters DRY by **extracting a component** (preferred)
  or a small semantic class built with `@apply` in the global stylesheet. Do
  not copy-paste the same long utility string across many files.
- Express interaction and responsive variants with Tailwind state/breakpoint
  prefixes (`hover:`, `focus-visible:`, `disabled:`, `sm:`, `md:`, `lg:`),
  chosen from real layout pressure, not device names.
- For conditional classes, use a tiny, explicit join helper (e.g. `clsx`-style)
  rather than nested string templates. Keep the conditions readable.
- Respect dark mode and theme tokens if the project configures them; do not
  hardcode hex colors when a token exists.

## Global CSS Is The Exception

Global CSS lives in the Tailwind entry stylesheet (e.g. `src/main.css` /
`src/index.css`) and is limited to:

- the Tailwind import/directives and theme configuration
- CSS reset/normalize beyond Tailwind's preflight, web-font `@font-face`
- a small set of `@apply`-based semantic classes for repeated clusters
- baseline body/root styles

Do not add feature-specific selectors or component-name selectors to global
CSS. Component-specific styling stays in the component's `className`.

## Inline Style Rule

Do not use inline `style={{ ... }}` for static styling that belongs in Tailwind
utilities. Reserve inline `style` for genuinely dynamic runtime values (e.g. a
width derived from measurement, a chart dimension, a CSS variable set from
data).

## Forbidden Patterns

- Two exported components in one `.tsx` file.
- Component filename and exported component name disagreeing.
- `client.data`, `RayfinClient`, or an auth SDK imported inside a component or
  page.
- Async orchestration, reducers, or repository calls inside a presentational
  component.
- Inline `style={{ ... }}` for static styling that belongs in a Tailwind class.
- Copy-pasted long utility strings instead of an extracted component or
  `@apply` class.
- Feature-specific selectors or component-name selectors added to global CSS.

## Verification

The component-file and styling audits — one component per file, matching file
and component names, no SDK imports in components, no static inline `style`,
no duplicated global selectors — live with the rest of the architecture checks
in [`verification-gates.md`](verification-gates.md). Treat matches as signals:
investigate each and either fix it or document why it is the deliberate
exception. Verify rendered results in a real browser per
[`playwright-ui-verification.md`](playwright-ui-verification.md).
