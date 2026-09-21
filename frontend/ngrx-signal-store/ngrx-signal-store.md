# NgRx Signal Store — Implementation Guide

> Detailed implementation reference for feature state management using `@ngrx/signals`. Read the [Frontend Architecture](../../frontend-architecture.md) for the decision of when to use Signal Store vs classic NgRx Store.

---

## Setup

```bash
npm install @ngrx/signals @ngrx/operators
```

No module registration needed. Signal stores are provided at the route level or with `providedIn: 'root'` for global stores.

---

## Anatomy of a Signal Store

A store is built by composing feature functions inside `signalStore()`:

```typescript
// queue.store.ts
import {
  signalStore, withState, withComputed, withMethods, withHooks, patchState
} from '@ngrx/signals';
import {
  withEntities, setAllEntities, addEntity, updateEntity, removeEntity
} from '@ngrx/signals/entities';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { inject, computed } from '@angular/core';
import { pipe, switchMap, tap } from 'rxjs';
import { QueueApiService } from '../services/queue-api.service';
import { QueueEntry, QueueStatus } from '../models/queue.models';

type QueueState = {
  businessId: string | null;
  queueStatus: QueueStatus;
  loading: boolean;
  error: string | null;
};

export const QueueStore = signalStore(
  { providedIn: 'root' },           // or omit and provide at route level

  withState<QueueState>({
    businessId: null,
    queueStatus: 'active',
    loading: false,
    error: null,
  }),

  withEntities<QueueEntry>(),       // adds entities(), ids(), entityMap() signals

  withComputed(({ entities, queueStatus }) => ({
    waitingEntries: computed(() => entities().filter(e => e.status === 'waiting')),
    waitingCount:   computed(() => entities().filter(e => e.status === 'waiting').length),
    isQueueOpen:    computed(() => queueStatus() === 'active'),
  })),

  withMethods((store, api = inject(QueueApiService)) => ({

    // Sync mutation
    setBusinessId(id: string): void {
      patchState(store, { businessId: id });
    },

    markServed(ticketId: string): void {
      patchState(store, updateEntity({ id: ticketId, changes: { status: 'served' } }));
    },

    // Async — Observable-based
    loadQueue: rxMethod<string>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(businessId =>
          api.getQueueEntries(businessId).pipe(
            tapResponse({
              next: entries => patchState(store, setAllEntities(entries), { loading: false }),
              error: (err: Error) => patchState(store, { error: err.message, loading: false }),
            })
          )
        )
      )
    ),

    addEntry: rxMethod<CreateQueueEntryRequest>(
      pipe(
        switchMap(request =>
          api.joinQueue(request).pipe(
            tapResponse({
              next: entry  => patchState(store, addEntity(entry)),
              error: (err: Error) => patchState(store, { error: err.message }),
            })
          )
        )
      )
    ),

    removeEntry(id: string): void {
      patchState(store, removeEntity(id));
    },
  })),

  withHooks({
    onInit(store) {
      // Runs once when the store is first injected
      console.log('QueueStore initialized');
    },
    onDestroy(store) {
      console.log('QueueStore destroyed');
    }
  })
);
```

---

## Providing the Store

### At Route Level (Preferred for Feature Stores)

```typescript
// queue.routes.ts
export const queueRoutes: Routes = [
  {
    path: '',
    providers: [QueueStore],      // Store lives only while this route is active
    children: [
      {
        path: '',
        loadComponent: () => import('./pages/queue-management-page.component')
          .then(m => m.QueueManagementPageComponent),
      },
    ]
  }
];
```

### Globally (for App-Wide Stores)

```typescript
// In the store definition:
export const AuthStore = signalStore({ providedIn: 'root' }, ...);
```

---

## Consuming in a Component

```typescript
@Component({
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (store.loading()) {
      <app-loading-spinner />
    }

    @for (entry of store.waitingEntries(); track entry.id) {
      <app-ticket-card
        [entry]="entry"
        (serve)="store.markServed($event)"
      />
    } @empty {
      <p class="text-gray-400 text-center py-8">Queue is empty</p>
    }
  `
})
export class QueueManagementPageComponent implements OnInit {
  store = inject(QueueStore);

  // Route param bound automatically via withComponentInputBinding()
  businessId = input.required<string>();

  ngOnInit() {
    this.store.loadQueue(this.businessId());
  }
}
```

---

## `withEntities` — Entity Collections

`withEntities<T>()` provides a normalized entity collection with built-in signals and update helpers.

**Auto-generated signals:**
- `store.entities()` — array of all entities in insertion order
- `store.ids()` — array of entity IDs
- `store.entityMap()` — `Record<string, T>` map

**Update helpers (used inside `patchState`):**

| Helper | Description |
|---|---|
| `setAllEntities(items)` | Replace the entire collection |
| `setEntities(items)` | Upsert multiple entities |
| `addEntity(item)` | Add a single entity |
| `updateEntity({ id, changes })` | Partial update by ID |
| `removeEntity(id)` | Remove by ID |
| `removeEntities(ids)` | Remove multiple by ID |

By default, entities must have an `id: string` field. For a different key:

```typescript
withEntities<QueueEntry, 'ticketId'>({ idKey: 'ticketId' })
```

---

## `rxMethod` — Async Operations

`rxMethod` wraps an RxJS pipeline into a store method. It handles Observable lifecycles and connects neatly to `tapResponse` from `@ngrx/operators`.

```typescript
// Accepts a static value, a signal, or an Observable
store.loadQueue('biz-123');                          // static
store.loadQueue(this.businessId);                    // signal — re-runs on change
store.loadQueue(this.businessId$);                   // Observable
```

Use `tapResponse` to handle success and error without breaking the stream:

```typescript
tapResponse({
  next: result => patchState(store, ...),
  error: (err: Error) => patchState(store, { error: err.message }),
})
```

---

## `patchState` — Mutating State

`patchState` is the only way to update store state. It performs a shallow merge — provide only the fields you want to change.

```typescript
// Single field
patchState(store, { loading: true });

// Multiple fields
patchState(store, { loading: false, error: null });

// Entity operation + state update in one call
patchState(store, setAllEntities(entries), { loading: false });

// Updater function for computed updates
patchState(store, state => ({ count: state.count + 1 }));
```

---

## Composing Stores

A store method can inject and call another store:

```typescript
withMethods((store, notif = inject(NotificationStore)) => ({
  loadQueue: rxMethod<string>(
    pipe(
      switchMap(id => api.getQueueEntries(id).pipe(
        tapResponse({
          next: entries => patchState(store, setAllEntities(entries)),
          error: (err: Error) => {
            patchState(store, { error: err.message });
            notif.showError(err.message);           // call another store's method
          },
        })
      ))
    )
  )
}))
```

---

## Custom Store Features

Reusable store slices (e.g., loading state, pagination) can be extracted into custom feature functions:

```typescript
// shared/store/with-loading.ts
export function withLoadingState() {
  return signalStoreFeature(
    withState({ loading: false, error: null as string | null }),
    withMethods(store => ({
      setLoading: () => patchState(store, { loading: true, error: null }),
      setError: (error: string) => patchState(store, { loading: false, error }),
      clearLoading: () => patchState(store, { loading: false }),
    }))
  );
}

// Used in any store:
export const QueueStore = signalStore(
  withLoadingState(),
  withEntities<QueueEntry>(),
  ...
);
```

---

## Testing a Signal Store

```typescript
describe('QueueStore', () => {
  let store: InstanceType<typeof QueueStore>;
  let apiSpy: jasmine.SpyObj<QueueApiService>;

  beforeEach(() => {
    apiSpy = jasmine.createSpyObj('QueueApiService', ['getQueueEntries']);

    TestBed.configureTestingModule({
      providers: [
        QueueStore,
        { provide: QueueApiService, useValue: apiSpy }
      ]
    });

    store = TestBed.inject(QueueStore);
  });

  it('loads queue entries', fakeAsync(() => {
    const mockEntries: QueueEntry[] = [
      { id: '1', ticketNumber: 1, status: 'waiting', position: 1, estimatedWait: 5 }
    ];
    apiSpy.getQueueEntries.and.returnValue(of(mockEntries));

    store.loadQueue('biz-1');
    tick();

    expect(store.entities()).toEqual(mockEntries);
    expect(store.loading()).toBe(false);
    expect(store.error()).toBeNull();
  }));

  it('computes waiting count correctly', () => {
    patchState(store, setAllEntities([
      { id: '1', status: 'waiting' },
      { id: '2', status: 'called' },
      { id: '3', status: 'waiting' },
    ] as QueueEntry[]));

    expect(store.waitingCount()).toBe(2);
  });

  it('marks an entry as served', () => {
    patchState(store, addEntity({ id: '1', status: 'waiting' } as QueueEntry));
    store.markServed('1');
    expect(store.entities()[0].status).toBe('served');
  });
});
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Mutating store state directly | Always use `patchState()` |
| Deriving state in a component | Move to `withComputed()` |
| `rxMethod` not re-running on signal change | Pass the signal itself, not `signal()` |
| Forgetting `tapResponse` error handler | Stream breaks on error without it |
| Providing the store globally when it's feature-specific | Use route-level `providers` instead |
