# Frontend Architecture

## Status

BetterF uses Angular with standalone components, signals, NgRx Signal Store, and Tailwind CSS. The inherited technology conventions are retained. Product routes, feature names, supported locales, and visual identity remain to be designed. The [frontend guides](frontend/README.md) explain implementation patterns. See the accepted [technology baseline](decisions/0001-technology-baseline.md).

## Technology stack

| Concern | Selected technology |
|---|---|
| Framework and language | Angular, TypeScript |
| Component model | Standalone components |
| Reactivity | Angular signals with RxJS interoperability |
| Feature state | NgRx Signal Store |
| Cross-feature state | Classic NgRx Store only where shared/complex flows justify it |
| Routing | Angular Router, lazy loading, functional guards |
| HTTP | Angular HttpClient and functional interceptors |
| Styling | Tailwind CSS |
| Localization | Transloco |
| Testing | Angular TestBed, Jest, Playwright |

Pin compatible versions in the application. The stack is selected; installation and configuration remain implementation work.

## Angular implementation conventions

- Bootstrap with `bootstrapApplication()` and central provider configuration in `app.config.ts`; do not introduce NgModule-based application structure.
- Use standalone components, directives, and pipes. Import each template dependency explicitly.
- Retain zoneless change detection and `ChangeDetectionStrategy.OnPush` as project conventions; verify supported bootstrap APIs for the selected Angular version.
- Use external HTML templates, `inject()` for dependency injection, and `input()`, `output()`, and `model()` for component interfaces.
- Use signals for local state and `computed()` for derived state; effects perform side effects only.
- Use `toSignal()` to expose Observable state to templates. Handle initial state explicitly and clean up manual subscriptions.
- Use block control flow (`@if`, `@for`, `@switch`) and a stable identity in `@for` tracking.
- Use NgRx Signal Store for feature state, `patchState()` for updates, entity helpers for normalized collections, and `rxMethod` for Observable-based operations.
- Keep one store per domain concept within a feature. Coordinate cross-store interactions at page/use-case boundaries rather than injecting feature stores into each other.
- Scope feature stores through route providers and verify reset/reuse behavior; reserve global providers for app-wide state.
- Keep presentational components independent of stores and services; pass data and translated labels through inputs.
- Use dedicated typed API clients consumed through store operations. Register functional HTTP interceptors centrally.
- Use Tailwind utility classes for component styling; keep shared styles in the global stylesheet rather than component-specific CSS files.
- Use Transloco for localized resources and signal-based consumption; the supported locales and persistence policy remain product decisions.

## Feature organization

Group code by feature or domain, with routes, page components, reusable components, state, API clients, and models close to their owner. Determine actual feature names from BetterF requirements.

Keep app-wide infrastructure distinct from feature logic. Share code when it has a demonstrated cross-feature responsibility; do not turn a shared directory into a dependency shortcut.

## Components and state

- Page components coordinate state, services, and user interactions.
- Presentational components receive data and emit user intent through explicit interfaces.
- Keep local UI state local; give shared and feature state an explicit owner and lifetime.
- Derive values from their source instead of maintaining duplicate state.
- Keep derived computations free of side effects. Perform asynchronous work through explicit operations with loading, success, and failure states.
- Keep store dependencies explicit and avoid circular relationships.

Follow the Angular conventions below and pin compatible package versions in the application.

## Routing and API integration

Choose routes from approved user journeys. Document which routes are public, which require authentication, and how navigation handles missing resources, access denial, and unsaved work.

Frontend guards improve navigation; the backend remains responsible for authorization.

Keep HTTP details in dedicated API clients. Use typed request and response models aligned with the server contract. Configure base URLs per environment. Centralize shared request concerns where appropriate, while keeping feature-specific errors visible to the feature.

Authentication transport, refresh behavior, error presentation, and any real-time connection strategy remain undecided. Do not assume a particular cookie, token, or endpoint contract.

## UI quality

Provide loading, empty, error, and permission states. Use semantic controls, keyboard-accessible interactions, visible focus, and responsive layouts. Share visual conventions consistently without inheriting another product's colors or status vocabulary.

Keep user-facing text separate from business logic where localization is needed. Supported languages, default locale, locale persistence, and right-to-left requirements must follow BetterF requirements.

## Testing

Test state transitions and API clients in isolation; test components through their public inputs and user interactions. Use end-to-end tests for approved critical journeys. Prefer semantic selectors or stable test identifiers over incidental CSS structure.

Use Angular TestBed and HTTP testing support, Jest for unit tests, and Playwright for end-to-end tests. Verify runner configuration against the selected Angular version. Exact test commands, coverage gates, browser support, and visual conventions will be recorded when the application is configured.
