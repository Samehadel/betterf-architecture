# Angular Signals — Implementation Guide

> Detailed implementation reference for Angular's reactivity primitives. Signals replace Zone.js-based change detection and Observable subscriptions for local and derived UI state.

---

## Core Primitives

### `signal()` — Writable State
### `computed()` — Derived State

**Rules:**
- `computed()` is **lazy** — it only recalculates when read after a dependency changed
- Never produce side effects inside `computed()` — use `effect()` for that
- Computed signals are **read-only** — you cannot call `.set()` on them

### `effect()` — Side Effects

```typescript
import { signal, effect, inject } from '@angular/core';

@Component({ ... })
export class QueueComponent {
  selectedId = signal<string | null>(null);

  constructor() {
    // Runs immediately, then re-runs whenever selectedId() changes
    effect(() => {
      const id = this.selectedId();
      if (id) {
        console.log(`Selected ticket: ${id}`);
        // e.g. sync to localStorage, trigger analytics, update document title
      }
    });
  }
}
```

**Rules:**
- `effect()` must be called inside an injection context (constructor or `runInInjectionContext`)
- Use `effect()` for side effects only — never for state derivation
- Effects automatically track which signals they read and re-run when those signals change
- An effect cannot write to a signal it reads unless you use `{ allowSignalWrites: true }` — avoid this pattern

---

## Component I/O

### `input()` — Signal-Based Inputs

```typescript
import { Component, input, computed } from '@angular/core';

@Component({ standalone: true, ... })
export class TicketCardComponent {
  // Required — throws if parent doesn't pass it
  ticket = input.required<QueueTicket>();

  // Optional with default
  highlighted = input(false);

  // With transform — converts the raw input value
  ticketNumber = input(0, { transform: (v: string | number) => Number(v) });

  // Computed from input signal
  waitLabel = computed(() =>
    `~${this.ticket().estimatedWait} min`
  );
}
```

Usage in a parent template:
```html
<app-ticket-card [ticket]="myTicket" [highlighted]="true" />
```

### `output()` — Signal-Based Outputs

```typescript
import { output } from '@angular/core';

@Component({ standalone: true, ... })
export class TicketCardComponent {
  ticket = input.required<QueueTicket>();

  served  = output<string>();    // emits the ticket ID
  noShow  = output<string>();

  onServe() {
    this.served.emit(this.ticket().id);
  }
}
```

Usage in parent:
```html
<app-ticket-card [ticket]="t" (served)="onServed($event)" />
```

### `model()` — Two-Way Binding

```typescript
import { model } from '@angular/core';

@Component({ standalone: true, ... })
export class NotesInputComponent {
  // Two-way bindable signal — parent can use [(notes)]="myNotes"
  notes = model('');
}
```

Usage in parent:
```html
<app-notes-input [(notes)]="ticketNotes" />
```

---

## RxJS Interop

### `toSignal()` — Observable → Signal

Converts an Observable (e.g. from NgRx selectors or HTTP) into a signal for use in templates and `computed()`.

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({ standalone: true, ... })
export class DashboardComponent {
  private store = inject(Store);

  // NgRx selector → signal
  notifications = toSignal(
    this.store.select(selectNotifications),
    { initialValue: [] }                      // required if stream is async
  );

  // HTTP observable → signal
  private http = inject(HttpClient);
  serverTime = toSignal(
    this.http.get<string>('/api/time'),
    { initialValue: null }
  );
}
```

**Options:**

| Option | Description |
|---|---|
| `initialValue` | Value before the Observable emits. Required for async streams. |
| `requireSync` | Asserts the Observable emits synchronously. Throws if it doesn't. |
| `injector` | Use when calling outside an injection context. |

### `toObservable()` — Signal → Observable

Converts a signal to an Observable when you need RxJS operators like `switchMap`, `debounceTime`, etc.

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';

@Component({ standalone: true, ... })
export class CustomerStatusPageComponent {
  token = input.required<string>();

  private statusService = inject(CustomerStatusService);

  // Re-fetch whenever token changes
  status = toSignal(
    toObservable(this.token).pipe(
      switchMap(token => this.statusService.getStatusStream(token))
    ),
    { initialValue: null }
  );
}
```

### `takeUntilDestroyed()` — Auto-Cleanup

Use instead of manual `ngOnDestroy` unsubscription:

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ ... })
export class QueueComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.wsService.messages$.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(event => ...);
  }
}
```

Note: `toSignal()` handles teardown automatically — you only need `takeUntilDestroyed` when subscribing manually.

---

## Signals in Templates

With `provideZonelessChangeDetection()`, templates update **only** when a signal they read changes. Access signals by calling them:

```html
<!-- Signal values — call like a function -->
<p>{{ queue().status }}</p>
<p>{{ waitingCount() }} customers waiting</p>

<!-- Computed signal -->
<p>{{ estimatedWait() }}</p>

<!-- Conditional on signal -->
@if (store.isQueueOpen()) {
  <button (click)="callNext()">Call Next</button>
}

<!-- Iterate signal array -->
@for (entry of store.waitingEntries(); track entry.id) {
  <app-ticket-card [entry]="entry" />
}
```

---

## Zoneless Change Detection

With `provideZonelessChangeDetection()`:
- Angular no longer patches browser APIs (setTimeout, promises, events)
- Change detection runs only when a **signal** changes or `markForCheck()` is called explicitly
- All components must use `ChangeDetectionStrategy.OnPush` — this is enforced by the zoneless scheduler

If you use a third-party library that mutates DOM without signals (e.g. a legacy chart library), you may need to manually trigger detection:

```typescript
private cdr = inject(ChangeDetectorRef);

onThirdPartyUpdate() {
  // After external mutation:
  this.cdr.markForCheck();
}
```

---

## Common Patterns

### Signal-Based Filter

```typescript
@Component({ standalone: true, ... })
export class QueueListComponent {
  store = inject(QueueStore);

  // Local UI state — not in the store
  filterStatus = signal<QueueEntryStatus | 'all'>('all');

  filteredEntries = computed(() => {
    const filter = this.filterStatus();
    const entries = this.store.entities();
    return filter === 'all' ? entries : entries.filter(e => e.status === filter);
  });
}
```

### Signal-Based Form State

```typescript
@Component({ standalone: true, imports: [ReactiveFormsModule], ... })
export class QueueSettingsComponent {
  form = new FormGroup({
    maxCapacity: new FormControl(50),
    isOpen: new FormControl(true),
  });

  // Expose form value as a signal
  formValue = toSignal(this.form.valueChanges, {
    initialValue: this.form.value
  });

  isFormDirty = toSignal(
    this.form.statusChanges.pipe(map(() => this.form.dirty)),
    { initialValue: false }
  );
}
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Side effects inside `computed()` | Move to `effect()` |
| Calling `effect()` outside injection context | Call in constructor or use `runInInjectionContext` |
| Missing `initialValue` in `toSignal()` for async streams | Component may receive `undefined` on first render |
| Circular signal dependencies in `effect()` | Restructure — a signal should not write to itself or its dependents |
