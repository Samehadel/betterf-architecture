# Angular Routing — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Route structure

Use Angular Router and define routes around approved user journeys. Use `loadComponent` for individual standalone pages and `loadChildren` to group feature route trees where useful. Include deliberate fallback and missing-resource behavior.

Illustrative route, not a BetterF URL:

```typescript
{
  path: 'items/:id',
  loadComponent: () => import('./item-page.component')
    .then(m => m.ItemPageComponent),
}
```

## Parameters and state

Use `withComponentInputBinding()` to expose route data as component inputs. Account for parameter changes while the same component instance remains active.

Choose provider scope based on state ownership. Verify route reuse, reset, and injector lifetime instead of assuming a fresh store is created on every navigation.

## Guards and resolvers

Functional guards can control navigation based on authentication or permissions. Backend authorization remains mandatory. Guard return values should express navigation decisions rather than mixing checks with unrelated effects.

Use a deactivation guard when unsaved changes require protection. A resolver can load required data before navigation; define loading and failure behavior so errors do not leave navigation unexplained.

## Navigation and accessibility

Use router links for navigable controls, provide an active-page indication, and preserve keyboard behavior. Handle post-login return destinations according to the accepted authentication flow and validate destinations before using them.

## Verification

Test direct links, reloads, parameter changes, unknown paths, denied access, unsaved changes, and the selected state lifetime. Actual routes, public pages, role checks, and landing destinations remain undefined until BetterF journeys are designed.
