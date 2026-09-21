# NgRx Signal Store — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Store composition

Use NgRx Signal Store for feature state. The following building blocks can organize feature state:

| Building block | Purpose |
|---|---|
| `withState` | Initial state |
| `withComputed` | Pure derived values |
| `withMethods` | Explicit operations and state changes |
| `withEntities` | Normalized collections |
| `withHooks` | Lifecycle behavior |
| `patchState` | State updates |
| `rxMethod` | Observable-based operations |

Illustrative local state:

```typescript
export const FilterStore = signalStore(
  withState({ query: '' }),
  withComputed(({ query }) => ({
    normalizedQuery: computed(() => query().trim().toLowerCase()),
  })),
  withMethods(store => ({
    setQuery(query: string) {
      patchState(store, { query });
    },
  })),
);
```

Imports and provider configuration depend on the selected package versions.

## Collections and asynchronous work

Use normalized collections where identity-based lookup and updates are useful. Keep derived values in computed state. Define loading, success, empty, and error behavior explicitly.

An asynchronous store method should call a typed API client, update state on success, and recover from failure without permanently terminating future requests. Choose RxJS operators according to cancellation and ordering requirements rather than applying one operator to every operation.

## Ownership and scope

Provide feature stores at a scope matching their intended lifetime. Reserve app-wide stores for shared application state. Do not assume route-scoped injection automatically resets a store on every navigation; verify reuse and teardown behavior.

Keep store dependencies explicit and avoid cycles. Coordinate multiple domain operations at a clear use-case or page boundary. Extract reusable state features when repeated behavior justifies it.

## Testing

Mock API boundaries and verify observable state transitions, derived values, errors, repeated requests, and intended lifecycle/reset behavior. Do not import another product's domain transitions as test requirements.
