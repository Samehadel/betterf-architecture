# Angular Signals — Reference Patterns

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## State and derived values

If Angular signals are adopted, use writable signals for local state and computed signals for values derived from it. Keep side effects out of computed values.

```typescript
const search = signal('');
const items = signal<{ id: string; name: string }[]>([]);
const filteredItems = computed(() =>
  items().filter(item => item.name.includes(search()))
);
```

Effects are for explicit side effects, such as synchronizing an external API. Avoid using effects to maintain duplicate derived state or creating circular dependencies. Observe the selected Angular version's injection-context and cleanup requirements.

## Component interfaces

Signal inputs, outputs, and models can express component contracts:

```typescript
readonly label = input.required<string>();
readonly selected = output<string>();
readonly expanded = model(false);
```

Read a signal by calling it in the template. Emit output events to report user intent rather than giving child components direct access to feature stores.

## RxJS interoperability

- `toSignal()` exposes Observable values as a signal. Choose an initial value or handle the state before the first emission.
- `toObservable()` makes signal changes available to RxJS operators.
- `takeUntilDestroyed()` can tie manual subscriptions to the component lifetime.

For changing resource identifiers, design data loading to respond to identifier changes and handle cancellation or stale responses. Do not assume initialization runs on every parameter change.

## Rendering and cleanup

Choose change detection and zoneless configuration when the Angular version is selected. Do not assume that signals are the only notification mechanism or that a particular change detection strategy is framework-mandated.

Test derived state, asynchronous initial states, input changes, and teardown of subscriptions/effects. Keep framework-version configuration in the application rather than treating this reference as a runnable setup.
