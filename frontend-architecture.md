# Frontend Architecture

## Status

This document retains reusable frontend design guidance. BetterF's frontend framework, rendering model, state library, styling system, routes, and supported locales are not yet established here. The [frontend guides](frontend/README.md) preserve optional Angular, NgRx, and Tailwind patterns; their presence does not select that stack.

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

Signal APIs, store scope, template conventions, dependency injection syntax, and change detection configuration depend on the selected framework and versions.

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

Frameworks, test commands, coverage targets, browser support, and visual conventions will be recorded when the application is configured.
