# Standalone Components — Implementation Guide

> Detailed implementation reference for Angular standalone components. There are no NgModules in this codebase — every component, directive, and pipe is standalone.

---

## Anatomy of a Standalone Component

```typescript
import { Component, ChangeDetectionStrategy, input, output, computed, inject } from '@angular/core';
import { DatePipe } from '@angular/common';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-ticket-card',
  standalone: true,                                  // Required
  changeDetection: ChangeDetectionStrategy.OnPush,   // Required — enforced by zoneless
  imports: [DatePipe, RouterLink],                   // Everything used in the template
  template: `
    <div class="rounded-lg border p-4">
      <p class="font-bold">#{{ ticket().ticketNumber }}</p>
      <p class="text-sm text-gray-500">{{ ticket().joinedAt | date:'shortTime' }}</p>
      <a [routerLink]="['/ticket', ticket().id]">Details</a>
      <button (click)="served.emit(ticket().id)">Mark Served</button>
    </div>
  `
})
export class TicketCardComponent {
  ticket = input.required<QueueTicket>();
  served = output<string>();
}
```

**Non-negotiable rules:**
- `standalone: true` on every component, directive, and pipe
- `changeDetection: ChangeDetectionStrategy.OnPush` on every component
- `imports` must list everything the template uses — Angular will not infer it
- `inject()` for all DI — not constructor parameters

---

## What Goes in `imports`

The `imports` array replaces what NgModules used to handle. You import exactly what the template references.

| Template usage | Import |
|---|---|
| `{{ value \| date }}` | `DatePipe` from `@angular/common` |
| `{{ value \| currency }}` | `CurrencyPipe` from `@angular/common` |
| `[routerLink]` | `RouterLink` from `@angular/router` |
| `<router-outlet>` | `RouterOutlet` from `@angular/router` |
| `[formControl]`, `[formGroup]` | `ReactiveFormsModule` from `@angular/forms` |
| `[(ngModel)]` | `FormsModule` from `@angular/forms` |
| `<app-ticket-card>` | `TicketCardComponent` |
| Custom pipe | `MyCustomPipe` |
| Custom directive | `MyCustomDirective` |

You **do not** need to import `AsyncPipe` — in this codebase we use `toSignal()` instead of `async` pipe.

---

## Smart vs Presentational Components

### Smart (Page-Level) Component

Connects to stores and services. Lives in `features/{domain}/pages/`.

```typescript
@Component({
  selector: 'app-queue-management-page',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [QueueTicketCardComponent, QueueStatsBarComponent, LoadingSpinnerComponent],
  template: `
    @if (store.loading()) {
      <app-loading-spinner />
    }

    <app-queue-stats-bar [waitingCount]="store.waitingCount()" />

    @for (entry of store.waitingEntries(); track entry.id) {
      <app-queue-ticket-card
        [entry]="entry"
        (served)="store.markServed($event)"
      />
    } @empty {
      <p class="text-center py-12 text-gray-400">Queue is empty</p>
    }
  `
})
export class QueueManagementPageComponent implements OnInit {
  store = inject(QueueStore);
  businessId = input.required<string>();    // Bound from route param

  ngOnInit() {
    this.store.loadQueue(this.businessId());
  }
}
```

### Presentational Component

No store or service injection. All data via `input()`. Lives in `features/{domain}/components/` or `shared/components/`.

```typescript
@Component({
  selector: 'app-queue-stats-bar',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="flex gap-6 rounded-lg bg-white p-4 shadow-sm">
      <div>
        <p class="text-2xl font-bold text-indigo-600">{{ waitingCount() }}</p>
        <p class="text-sm text-gray-500">Waiting</p>
      </div>
    </div>
  `
})
export class QueueStatsBarComponent {
  waitingCount = input.required<number>();
}
```

---

## Shared Components

Shared components live in `shared/components/`. They are purely presentational, have no domain knowledge, and are imported wherever needed.

```
shared/components/
    ├── button/
    │   └── button.component.ts
    ├── badge/
    │   └── badge.component.ts
    ├── loading-spinner/
    │   └── loading-spinner.component.ts
    ├── modal/
    │   └── modal.component.ts
    ├── error-banner/
    │   └── error-banner.component.ts
    └── empty-state/
        └── empty-state.component.ts
```

Example — reusable `BadgeComponent`:

```typescript
@Component({
  selector: 'app-badge',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <span class="rounded-full px-3 py-1 text-xs font-medium"
          [class]="colorClass()">
      {{ label() }}
    </span>
  `
})
export class BadgeComponent {
  label  = input.required<string>();
  color  = input<'green' | 'yellow' | 'blue' | 'gray'>('gray');

  colorClass = computed(() => ({
    green:  'bg-green-100 text-green-800',
    yellow: 'bg-yellow-100 text-yellow-800',
    blue:   'bg-blue-100 text-blue-800',
    gray:   'bg-gray-100 text-gray-700',
  }[this.color()]));
}
```

---

## Standalone Directives and Pipes

Directives and pipes are also standalone and must be imported individually.

### Custom Pipe

```typescript
// wait-time.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'waitTime',
  standalone: true,
  pure: true,              // Default — recalculates only when input reference changes
})
export class WaitTimePipe implements PipeTransform {
  transform(minutes: number): string {
    if (minutes < 1) return 'Less than a minute';
    if (minutes === 1) return '~1 min wait';
    return `~${minutes} min wait`;
  }
}

// Usage in any component:
imports: [WaitTimePipe]

// In template:
{{ entry().estimatedWait | waitTime }}
```

### Custom Directive

```typescript
// auto-focus.directive.ts
import { Directive, ElementRef, OnInit, inject } from '@angular/core';

@Directive({
  selector: '[appAutoFocus]',
  standalone: true,
})
export class AutoFocusDirective implements OnInit {
  private el = inject(ElementRef);

  ngOnInit() {
    this.el.nativeElement.focus();
  }
}

// Usage:
imports: [AutoFocusDirective]

// In template:
<input appAutoFocus type="text" />
```

---

## Dependency Injection with `inject()`

Use `inject()` in the class field initializer or constructor body. Do not use constructor parameters.

```typescript
// ✅ Correct
@Component({ ... })
export class MyComponent {
  private store = inject(QueueStore);
  private router = inject(Router);
  private route  = inject(ActivatedRoute);
}

// ❌ Avoid
@Component({ ... })
export class MyComponent {
  constructor(
    private store: QueueStore,
    private router: Router
  ) {}
}
```

`inject()` can be called in any function executed within an injection context, which includes:
- Class field initializers
- `constructor()` body
- `runInInjectionContext(injector, fn)`

---

## Testing Standalone Components

Standalone components are imported directly into `TestBed` — no `declarations` array.

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { QueueTicketCardComponent } from './queue-ticket-card.component';

describe('QueueTicketCardComponent', () => {
  let fixture: ComponentFixture<QueueTicketCardComponent>;
  let component: QueueTicketCardComponent;

  const mockEntry: QueueEntry = {
    id: 'e1',
    ticketNumber: 5,
    status: 'waiting',
    position: 2,
    estimatedWait: 10,
    joinedAt: new Date().toISOString(),
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [QueueTicketCardComponent],    // Import standalone component directly
    }).compileComponents();

    fixture = TestBed.createComponent(QueueTicketCardComponent);
    component = fixture.componentInstance;

    // Set input() signals using setInput
    fixture.componentRef.setInput('entry', mockEntry);
    fixture.detectChanges();
  });

  it('should display ticket number', () => {
    const el = fixture.nativeElement.querySelector('[data-test="ticket-number"]');
    expect(el.textContent.trim()).toBe('#5');
  });

  it('should emit served event on click', () => {
    const spy = jest.fn();
    component.served.subscribe(spy);

    fixture.nativeElement.querySelector('[data-test="serve-btn"]').click();

    expect(spy).toHaveBeenCalledWith('e1');
  });

  it('should compute wait label', () => {
    expect(component.waitLabel()).toBe('Position 2 — ~10 min wait');
  });
});
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Forgetting `standalone: true` | Linter rule enforces it — add `@angular-eslint/prefer-standalone` |
| Importing a module instead of the specific class | Import `DatePipe`, not `CommonModule` |
| Using `declarations` in TestBed | Use `imports` instead |
| Missing `changeDetection: OnPush` | Required for zoneless — add to every component |
| Using constructor injection | Use `inject()` field initializer |
| `*ngIf` / `*ngFor` still imported via `CommonModule` | Remove `CommonModule`, use `@if` / `@for` blocks |
