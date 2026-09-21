# Frontend Implementation Guides

> These guides contain the detailed implementation patterns, code examples, and conventions for each technical area of the frontend. They are the **"how"** — the [Frontend Architecture](../frontend-architecture.md) is the **"what and why"**.

---

## Guides

| Topic | Description |
|---|---|
| [NgRx Signal Store](./ngrx-signal-store/ngrx-signal-store.md) | Defining stores, `withState`, `withEntities`, `withMethods`, `rxMethod`, `patchState`, testing |
| [Angular Signals](./angular-signals/angular-signals.md) | `signal`, `computed`, `effect`, `input`, `output`, `model`, `toSignal`, `toObservable`, zoneless |
| [Standalone Components](./standalone-components/standalone-components.md) | Component anatomy, `imports`, smart vs presentational, directives, pipes, DI with `inject()` |
| [Routing](./routing/routing.md) | `loadComponent`, `loadChildren`, route params as `input()`, functional guards, resolvers, route-level providers |
| [HTTP & Interceptors](./http-interceptors/http-interceptors.md) | Functional interceptors, API service pattern, typed request/response models, error handling, `HttpTestingController` |
| [Tailwind Styling](./tailwind-styling/tailwind-styling.md) | Setup, status badges, layout patterns, mobile-first, responsive prefixes, `@apply` |
| [Testing](./testing/testing.md) | Signal Store tests, standalone component tests, `HttpTestingController`, Playwright E2E |
| [i18n](./i18n/i18n.md) | Transloco setup, key naming convention (`feature.component.element`), signals-based usage, RTL with Tailwind, adding a new language |

---

## How to Use These Guides

- **Starting a new feature?** Read [Standalone Components](./standalone-components/standalone-components.md) and [NgRx Signal Store](./ngrx-signal-store/ngrx-signal-store.md) first.
- **Adding an API call?** Read [HTTP & Interceptors](./http-interceptors/http-interceptors.md).
- **Writing tests?** Read [Testing](./testing/testing.md).
- **Unsure about styling conventions?** Read [Tailwind Styling](./tailwind-styling/tailwind-styling.md).
