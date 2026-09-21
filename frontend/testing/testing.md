# Testing — Implementation Guide

> Detailed reference for unit testing Signal Stores and standalone components, and E2E testing with Playwright.

---

## Setup

```bash
# Jest for unit tests
npm install --save-dev jest @types/jest jest-preset-angular

# Playwright for E2E
npm install --save-dev @playwright/test
npx playwright install
```

```typescript
// jest.config.ts
export default {
  preset: 'jest-preset-angular',
  setupFilesAfterFramework: ['<rootDir>/setup-jest.ts'],
  testPathPattern: 'src/.*\\.spec\\.ts$',
  collectCoverageFrom: [
    'src/app/**/*.ts',
    '!src/app/**/*.routes.ts',
    '!src/app/**/*.models.ts',
  ],
  coverageThreshold: {
    global: { lines: 80, branches: 80 }
  }
};
```

---

## Unit Testing — Signal Stores

Test stores in isolation by mocking the API service. Verify state changes via signal reads.

```typescript
// features/queue/store/queue.store.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { of, throwError } from 'rxjs';
import { QueueStore } from './queue.store';
import { QueueApiService } from '../services/queue-api.service';
import { patchState } from '@ngrx/signals';
import { setAllEntities, addEntity } from '@ngrx/signals/entities';

describe('QueueStore', () => {
  let store: InstanceType<typeof QueueStore>;
  let apiSpy: jest.Mocked<QueueApiService>;

  const mockEntries: QueueEntry[] = [
    { id: '1', ticketNumber: 1, status: 'waiting', position: 1, estimatedWait: 5 },
    { id: '2', ticketNumber: 2, status: 'waiting', position: 2, estimatedWait: 10 },
    { id: '3', ticketNumber: 3, status: 'called',  position: 0, estimatedWait: 0 },
  ] as QueueEntry[];

  beforeEach(() => {
    apiSpy = {
      getQueueEntries: jest.fn(),
      joinQueue: jest.fn(),
    } as unknown as jest.Mocked<QueueApiService>;

    TestBed.configureTestingModule({
      providers: [
        QueueStore,
        { provide: QueueApiService, useValue: apiSpy }
      ]
    });

    store = TestBed.inject(QueueStore);
  });

  describe('loadQueue', () => {
    it('loads entries and clears loading state', fakeAsync(() => {
      apiSpy.getQueueEntries.mockReturnValue(of(mockEntries));

      store.loadQueue('biz-1');
      tick();

      expect(store.entities()).toEqual(mockEntries);
      expect(store.loading()).toBe(false);
      expect(store.error()).toBeNull();
    }));

    it('sets loading true while fetching', fakeAsync(() => {
      apiSpy.getQueueEntries.mockReturnValue(of(mockEntries).pipe(delay(100)));

      store.loadQueue('biz-1');
      expect(store.loading()).toBe(true);   // Before tick

      tick(100);
      expect(store.loading()).toBe(false);  // After response
    }));

    it('sets error on failure', fakeAsync(() => {
      apiSpy.getQueueEntries.mockReturnValue(
        throwError(() => ({ message: 'Network error' }))
      );

      store.loadQueue('biz-1');
      tick();

      expect(store.loading()).toBe(false);
      expect(store.error()).toBe('Network error');
      expect(store.entities()).toHaveLength(0);
    }));
  });

  describe('computed signals', () => {
    beforeEach(() => {
      patchState(store, setAllEntities(mockEntries));
    });

    it('computes waitingCount correctly', () => {
      expect(store.waitingCount()).toBe(2);
    });

    it('computes waitingEntries correctly', () => {
      expect(store.waitingEntries()).toHaveLength(2);
      expect(store.waitingEntries().every(e => e.status === 'waiting')).toBe(true);
    });
  });

  describe('markServed', () => {
    it('updates entry status to served', () => {
      patchState(store, addEntity({ id: '1', status: 'called' } as QueueEntry));
      store.markServed('1');
      expect(store.entities()[0].status).toBe('served');
    });
  });
});
```

---

## Unit Testing — Standalone Components

Import standalone components directly into `TestBed.configureTestingModule({ imports: [...] })`. Set signal inputs with `fixture.componentRef.setInput()`.

```typescript
// features/queue/components/queue-ticket-card/queue-ticket-card.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { QueueTicketCardComponent } from './queue-ticket-card.component';
import { By } from '@angular/platform-browser';

describe('QueueTicketCardComponent', () => {
  let fixture: ComponentFixture<QueueTicketCardComponent>;
  let component: QueueTicketCardComponent;

  const mockEntry: QueueEntry = {
    id: 'e1',
    ticketNumber: 7,
    customerName: 'Ahmed Al-Rashid',
    serviceType: 'Haircut',
    status: 'waiting',
    position: 3,
    estimatedWait: 15,
    joinedAt: new Date().toISOString(),
    calledAt: null,
    servedAt: null,
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [QueueTicketCardComponent],   // Standalone — import directly
    }).compileComponents();

    fixture = TestBed.createComponent(QueueTicketCardComponent);
    component = fixture.componentInstance;

    fixture.componentRef.setInput('entry', mockEntry);
    fixture.detectChanges();
  });

  it('renders ticket number', () => {
    const el = fixture.debugElement.query(By.css('[data-test="ticket-number"]'));
    expect(el.nativeElement.textContent.trim()).toBe('#7');
  });

  it('renders customer name', () => {
    const el = fixture.debugElement.query(By.css('[data-test="customer-name"]'));
    expect(el.nativeElement.textContent).toContain('Ahmed Al-Rashid');
  });

  it('emits called event with ticket ID on Call button click', () => {
    const spy = jest.fn();
    component.called.subscribe(spy);

    fixture.debugElement.query(By.css('[data-test="call-btn"]')).nativeElement.click();
    fixture.detectChanges();

    expect(spy).toHaveBeenCalledWith('e1');
  });

  it('emits noShow event with ticket ID', () => {
    const spy = jest.fn();
    component.noShow.subscribe(spy);

    fixture.debugElement.query(By.css('[data-test="no-show-btn"]')).nativeElement.click();

    expect(spy).toHaveBeenCalledWith('e1');
  });

  it('computes positionLabel correctly', () => {
    expect(component.positionLabel()).toBe('Position 3 — ~15 min wait');
  });

  it('shows correct status badge class for waiting status', () => {
    expect(component.statusClass()).toContain('yellow');
  });
});
```

---

## Unit Testing — Services with `HttpTestingController`

```typescript
// features/queue/services/queue-api.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { QueueApiService } from './queue-api.service';

describe('QueueApiService', () => {
  let service: QueueApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        QueueApiService,
        provideHttpClient(),
        provideHttpClientTesting(),
      ]
    });

    service = TestBed.inject(QueueApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());   // Ensures no unexpected requests

  it('GET /api/queues/:id/entries', () => {
    const mockEntries = [{ id: '1' }] as QueueEntry[];

    service.getQueueEntries('biz-1').subscribe(result => {
      expect(result).toEqual(mockEntries);
    });

    const req = httpMock.expectOne('/api/queues/biz-1/entries');
    expect(req.request.method).toBe('GET');
    req.flush(mockEntries);
  });

  it('POST /api/queues/:id/join', () => {
    const request: CreateQueueEntryRequest = {
      businessId: 'biz-1',
      customerPhone: '+966501234567',
      serviceType: 'haircut',
    };

    service.joinQueue(request).subscribe();

    const req = httpMock.expectOne('/api/queues/biz-1/join');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(request);
    req.flush({ id: 'new-1' });
  });

  it('handles 500 error', () => {
    let error: unknown;
    service.getQueueEntries('biz-1').subscribe({ error: e => error = e });

    httpMock.expectOne('/api/queues/biz-1/entries').flush(
      { message: 'Server error' },
      { status: 500, statusText: 'Internal Server Error' }
    );

    expect(error).toBeTruthy();
  });
});
```

---

## Testing Smart Components with Store

For page components that use a store, provide a mock store or a real store with mocked API service.

```typescript
// features/queue/pages/queue-management-page.component.spec.ts
describe('QueueManagementPageComponent', () => {
  let fixture: ComponentFixture<QueueManagementPageComponent>;
  let store: InstanceType<typeof QueueStore>;

  beforeEach(async () => {
    const apiSpy = {
      getQueueEntries: jest.fn().mockReturnValue(of([])),
    } as unknown as jest.Mocked<QueueApiService>;

    await TestBed.configureTestingModule({
      imports: [QueueManagementPageComponent],
      providers: [
        QueueStore,
        { provide: QueueApiService, useValue: apiSpy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(QueueManagementPageComponent);
    fixture.componentRef.setInput('businessId', 'biz-1');
    store  = TestBed.inject(QueueStore);
    fixture.detectChanges();
  });

  it('shows empty state when queue is empty', () => {
    const empty = fixture.debugElement.query(By.css('[data-test="empty-state"]'));
    expect(empty).toBeTruthy();
  });

  it('shows tickets when store has waiting entries', fakeAsync(() => {
    patchState(store, setAllEntities([
      { id: '1', ticketNumber: 1, status: 'waiting', position: 1, estimatedWait: 5 } as QueueEntry
    ]));
    tick();
    fixture.detectChanges();

    const cards = fixture.debugElement.queryAll(By.css('[data-test="ticket-card"]'));
    expect(cards).toHaveLength(1);
  }));
});
```

---

## `data-test` Attribute Convention

All test selectors use `data-test` attributes — never CSS classes or element types.

```html
<!-- In templates — add data-test to all interactive/important elements -->
<div data-test="queue-ticket-card">
  <span data-test="ticket-number">#{{ entry().ticketNumber }}</span>
  <span data-test="customer-name">{{ entry().customerName }}</span>
  <button data-test="call-btn" (click)="called.emit(entry().id)">Call</button>
  <button data-test="no-show-btn" (click)="noShow.emit(entry().id)">No Show</button>
</div>
```

```typescript
// In tests
fixture.debugElement.query(By.css('[data-test="call-btn"]')).nativeElement.click();
```

---

## E2E Testing with Playwright

E2E tests cover the critical business flows of the WhatsApp Virtual Queue.

```typescript
// e2e/queue-management.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Queue Management', () => {

  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-test="email-input"]', 'operator@example.com');
    await page.fill('[data-test="password-input"]', 'password');
    await page.click('[data-test="login-btn"]');
    await page.waitForURL('**/dashboard');
  });

  test('operator sees live queue and can call next customer', async ({ page }) => {
    await page.goto('/queue/biz-123');

    await expect(page.locator('[data-test="queue-ticket-card"]')).toHaveCount(3);
    await page.click('[data-test="call-next-btn"]');
    await expect(page.locator('[data-test="called-badge"]').first()).toBeVisible();
  });

  test('operator can mark customer as no-show', async ({ page }) => {
    await page.goto('/queue/biz-123');
    await page.locator('[data-test="no-show-btn"]').first().click();
    await expect(page.locator('[data-test="queue-ticket-card"]')).toHaveCount(2);
  });

});
```

```typescript
// e2e/customer-status.spec.ts
test.describe('Customer Status Page', () => {

  test('displays queue position for valid token', async ({ page }) => {
    await page.goto('/status/valid-token-abc123');

    await expect(page.locator('[data-test="queue-position"]')).toBeVisible();
    await expect(page.locator('[data-test="estimated-wait"]')).toBeVisible();
  });

  test('shows closed message when queue is closed', async ({ page }) => {
    await page.goto('/status/token-for-closed-queue');
    await expect(page.locator('[data-test="queue-closed-banner"]')).toBeVisible();
  });

  test('page is usable on mobile viewport', async ({ page }) => {
    await page.setViewportSize({ width: 390, height: 844 });  // iPhone 14
    await page.goto('/status/valid-token-abc123');
    await expect(page.locator('[data-test="queue-position"]')).toBeVisible();
  });

});
```

---

## Coverage Targets

| Layer | Target |
|---|---|
| Signal Store (state, computed, methods) | 90%+ |
| API services | 90%+ |
| Pipes | 100% |
| Presentational components | 80%+ |
| Page components | 70%+ |
| Guards / interceptors | 80%+ |

Run coverage:
```bash
npx jest --coverage
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `declarations` in TestBed for standalone components | Use `imports` |
| Setting `@Input()` directly: `component.ticket = ...` | Use `fixture.componentRef.setInput('ticket', ...)` |
| Selecting by CSS class in tests | Use `[data-test="..."]` attributes |
| Not calling `fixture.detectChanges()` after signal changes | Always call after state mutations |
| No `afterEach(() => httpMock.verify())` | Unexpected requests go undetected |
| E2E tests for every button click | E2E tests cover flows, not individual interactions |
