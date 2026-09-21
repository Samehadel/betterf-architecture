# Frontend Architecture & Design

**Template Version** — Customize all `[PLACEHOLDER]` sections below for your project.

> This document defines how the frontend is structured, how features relate to each other, and conventions every contributor must follow. It should complement your backend architecture.

---

## Overview

The frontend is a **[FRONTEND_FRAMEWORK]** application using **[COMPONENT_MODEL]** and **[STATE_MANAGEMENT]**. It follows **domain-driven design**, organized around the core business domains of the product.

### Stack

| Concern | Technology |
|---|---|
| Runtime | [FRONTEND_RUNTIME] |
| Component model | [COMPONENT_MODEL] |
| Reactivity | [REACTIVITY_PRIMITIVE] |
| State management | [STATE_MANAGEMENT_TOOL] |
| Styling | [CSS_FRAMEWORK] |
| Routing | [ROUTING_FRAMEWORK] |
| HTTP | [HTTP_CLIENT] |
| Testing | [TESTING_FRAMEWORKS] |

---

## App Bootstrap

There are **no NgModules**. The app bootstraps via `bootstrapApplication()` with a central `app.config.ts`. All providers — router, HTTP, store — are registered there using `provide*()` functions.

Key provider decisions:
- `provideZonelessChangeDetection()` — Zone.js is removed; change detection is driven entirely by signals
- `provideRouter()` with `withComponentInputBinding()` — route params bind directly to `input()` signals on components
- `provideHttpClient()` with `withInterceptors([...])` — functional interceptors registered at bootstrap
- NgRx store and effects are registered empty at root; feature stores self-register via route `providers`

---

## Standalone Components

Every component, directive, and pipe is standalone — they declare their own `imports` and are used directly without a module wrapper. There are no `NgModule` declarations anywhere in the codebase.

**Rules:**
- Every component has `standalone: true` and `changeDetection: ChangeDetectionStrategy.OnPush`
- Every component uses `templateUrl` pointing to an external `.html` file — never inline `template:` strings
- Components import only what their template actually uses
- Use `inject()` for dependency injection — not constructor injection
- Use `input()` / `output()` / `model()` signal APIs — not `@Input()` / `@Output()` decorators

---

## Angular Signals

Signals are the **primary reactivity primitive**, replacing Observable-based subscriptions for all local and derived UI state.

| Primitive | Purpose |
|---|---|
| `signal()` | Writable local state |
| `computed()` | Derived state — memoized, recalculates only when dependencies change |
| `effect()` | Side effects that react to signal changes (logging, DOM sync) |
| `input()` | Component input bound as a signal |
| `output()` | Component event emitter — replaces `EventEmitter` |
| `model()` | Two-way binding signal — replaces `[(ngModel)]` |
| `toSignal()` | Wraps an Observable into a signal for use in templates |
| `toObservable()` | Converts a signal to an Observable when needed by RxJS operators |

**Rules:**
- Prefer `signal()` + `computed()` over `BehaviorSubject` for local state
- Use `computed()` for derived values only — never put side effects inside it
- Use `effect()` only for side effects, not state derivation
- Use `toSignal()` to consume store selectors or HTTP streams in templates — no `async` pipe
- Never use `async` pipe — signals replace it entirely

---

## Template Syntax

Use Angular 17+ block syntax. Legacy structural directives (`*ngIf`, `*ngFor`, `*ngSwitch`) are not used.

| Old | New |
|---|---|
| `*ngIf="x"` | `@if (x) { }` |
| `*ngFor="let i of list"` | `@for (i of list; track i.id) { }` |
| `*ngSwitch` | `@switch (x) { @case (...) }` |
| — | `@empty { }` block inside `@for` for empty states |
| — | `@let x = expr` for template-local variables |

**Rules:**
- Always provide a `track` expression in `@for` using a unique, stable key — never `$index`
- Use `@empty` to handle empty collections inline — avoids an extra `@if` wrapper
- Never mix new block syntax with legacy structural directives in the same component

---

## State Management

### NgRx Signal Store — Default for Feature State

The NgRx Signal Store (`@ngrx/signals`) is the standard for all feature-level state. It collapses the classic actions/reducers/effects/selectors pattern into a single, composable store definition.

A feature store is built from composable pieces:
- `withState()` — defines the state shape and initial values
- `withEntities()` — normalized entity collection (replaces manual array management)
- `withComputed()` — derived signals computed from state
- `withMethods()` — synchronous mutations (`patchState`) and async operations (`rxMethod`)
- `withHooks()` — lifecycle hooks (`onInit`, `onDestroy`)

Feature stores are provided at the **route level**, not globally, so they only exist while the route is active.

**Rules:**
- Use `patchState()` to mutate store state — never set signals directly
- Use `withEntities` for any normalized collection ([DOMAIN_1], [DOMAIN_2], etc.)
- Use `rxMethod` for Observable-based async operations (HTTP calls and other asynchronous flows)
- Use `computed()` for derived values — never derive in components what can be computed in the store
- **One store per domain concept per feature.** A page managing `[DOMAIN_1]` and `[DOMAIN_2]` uses `[DOMAIN_1]Store` and `[DOMAIN_2]Store` — never a single combined store. Stores must not inject each other. The page component coordinates cross-store interactions.

### NgRx Store (Classic) — Cross-Feature and Complex Flows

Use the classic NgRx Store (actions, reducers, effects, selectors) only for:
- **Global state** shared across multiple unrelated features (e.g., auth session, notifications)
- **Complex async workflows** with many interdependent branching effects
- Migrating existing code before converting to Signal Store

In templates, consume classic NgRx selectors via `toSignal()` — no `select()` + `async` pipe.

---

## Component Pattern

Components follow a **smart / presentational** split:

**Smart (page-level) components** — connect to stores and services, coordinate data flow, live in `pages/`. They are the only components that inject feature stores or services.

**Presentational components** — receive all data via `input()`, emit events via `output()`, have no knowledge of stores or services. They are fully reusable and trivially testable.

**Rules:**
- Page components may inject stores and services; presentational components must not
- Never pass the store itself to a child — pass only the derived data it needs
- All components use `ChangeDetectionStrategy.OnPush` — required for zoneless

---

## Routing

Routes use `[ROUTE_LOADING_PATTERN]` for individual pages and grouping sub-routes. Feature stores and services are registered in the route `providers` array, not globally.

```
/[PUBLIC_ROUTE_1]         → [PAGE_COMPONENT_1]                  (public)
/[AUTH_ROUTE_1]           → [PAGE_COMPONENT_2]                  (auth required)
/[DOMAIN_1]/:id/...       → [DOMAIN_1] feature routes           (auth required)
/[DOMAIN_2]/...           → [DOMAIN_2] feature routes           (auth required)
```

**Rules:**
- Use [ROUTE_LOADING_PATTERN] for pages; use module/children grouping only when a feature has multiple sub-routes
- Route params are bound directly to [PARAM_BINDING_METHOD]
- Register feature stores via route `providers` — they load only when the route is active
- Guards are [GUARD_TYPE] — [GUARD_DESCRIPTION]

---

## HTTP Communication

All API calls go through **dedicated API services** — never from components directly. HTTP interceptors handle auth and error handling globally.

**API services** are pure HTTP clients: they construct URLs, set headers, and return typed Observables. No business logic lives there.

**Functional interceptors** (not class-based) handle:
- `errorInterceptor` — handles 401 (redirect to login) and other errors (show notification)

**Rules:**
- API services return `[HTTP_RETURN_TYPE]` — stores consume them via [STATE_CONSUMPTION_METHOD]
- All HTTP calls happen inside store methods — never triggered from components
- Use strongly-typed request and response models for all API shapes
- **One API service per backend resource.** `[DOMAIN_1]` endpoints live in `[DOMAIN_1]ApiService`; `[DOMAIN_2]` endpoints live in `[DOMAIN_2]ApiService`. Never mix endpoints for two distinct resources in one service.

---

## Feature Domains

```
features/
    ├── [DOMAIN_1]/        [DOMAIN_1 description]
    ├── [DOMAIN_2]/        [DOMAIN_2 description]
    ├── [DOMAIN_3]/        [DOMAIN_3 description]
    ├── auth/              Login, session, token management
    └── [GLOBAL_FEATURE]/  [Global feature description]
```

Each feature owns its own routes, store, API service, components, and models. Features do not import from each other — shared code goes in `shared/`.

### Domain Folder Structure

Each feature follows the same internal layout:

```
features/{domain}/
    ├── {domain}.routes.ts
    ├── store/
    │   └── {domain}.store.ts       ← [STATE_MANAGEMENT_STORE_TYPE]
    ├── services/
    │   └── {domain}-api.service.ts
    ├── models/
    │   └── {domain}.models.ts
    ├── components/                 ← domain-specific presentational components
    └── pages/                      ← routable smart components
```

---

## Core Services

App-wide singletons live in `core/` and use `providedIn: 'root'`. There is no `CoreModule`.

```
core/
    ├── interceptors/    error.interceptor.ts  (functional)
    ├── guards/          auth.guard.ts  (functional CanActivateFn)
    └── services/        auth.service.ts, notification.service.ts, logger.service.ts
```

---

## Shared Components

Reusable UI building blocks used across multiple features. No business logic, no store access.

```
shared/
    ├── components/    button, badge, modal, loading-spinner, error-banner, empty-state
    ├── directives/    auto-focus
    └── pipes/         relative-time ("2 min ago"), wait-time ("~5 min wait")
```

---

## Styling

All styling uses **[CSS_FRAMEWORK]** utility classes directly in templates. No component-scoped CSS files.

**Rules:**
- Use your CSS framework's utilities only — no per-component CSS files
- Use conditional class binding patterns appropriate to your framework
- For complex conditional class logic, compute a class string in a `computed()` signal (or equivalent)
- Use responsive design patterns for mobile-first layouts
- [PROJECT-SPECIFIC_STYLING_RULES]

---

## Testing

- **Unit tests** — test stores, services, and computed values in isolation with mocked dependencies
- **Component tests** — test components with their dependencies; set component inputs via your testing framework's methods
- **E2E tests** — cover critical user flows; use your E2E framework (e.g., [E2E_FRAMEWORK])

Use semantic selectors for all test assertions — never rely on CSS classes or element structure.

Maintain **[FRONTEND_COVERAGE_TARGET]%+ coverage** on stores, services, and computed values.

---

## What Goes Where — Quick Reference

| Thing | Where it lives |
|---|---|
| App providers, interceptors, router | [APP_BOOTSTRAP_FILE] |
| Route definitions | [ROUTE_DEFINITION_LOCATION] |
| Feature state | [FEATURE_STATE_LOCATION] |
| Cross-feature / global state | [GLOBAL_STATE_LOCATION] |
| HTTP client | `features/{domain}/services/{domain}-api.service.ts` |
| Routable page component | `features/{domain}/pages/` |
| Domain-specific component | `features/{domain}/components/` |
| Reusable UI component | `shared/components/` |
| App-wide singleton service | `core/services/` |
| HTTP interceptor | `core/interceptors/` |
| Route guard | `core/guards/` |
| Styling | [CSS_FRAMEWORK] in templates |
| Unit tests | `*.spec.ts` alongside source file |

---

## Things to Avoid

- **Never use module-based patterns** — use standalone components / modules only [CUSTOMIZE based on your framework]
- **Never use deprecated decorator-based APIs** — use functional / signal APIs (e.g., [PREFERRED_API_PATTERN])
- **Never use async patterns in templates** — use signals or equivalent reactive patterns
- **Never use legacy control-flow syntax** — use modern block syntax (e.g., [MODERN_CONTROL_FLOW])
- **Never mutate state outside store methods** — state is immutable outside designated mutation points
- **Never put side effects in computed/derived values** — use side-effect-specific mechanisms (`effect()`, `rxMethod`, etc.)
- **Never inject stores or services in presentational components** — they receive data only via `input()`
- **Never use class-based dependency injection** — use functional injection pattern (e.g., `inject()`)
- **Never call HTTP methods from components** — all HTTP calls go through API services consumed by stores
- **Never use inline templates in component decorators** — always use external template files
- **Never write component-scoped CSS files** — use your CSS framework in templates only
- **Never skip `track` in `@for`** — always use a unique stable key
