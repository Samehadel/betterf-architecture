# HTTP & Interceptors — Implementation Guide

> Detailed implementation reference for Angular `HttpClient`, functional interceptors, and the API service pattern.

---

## Setup

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { credentialsInterceptor } from './core/interceptors/credentials.interceptor';
import { loadingInterceptor } from './core/interceptors/loading.interceptor';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([credentialsInterceptor, loadingInterceptor, authInterceptor, errorInterceptor])
    ),
  ]
};
```

Interceptor order matters:

1. **`credentialsInterceptor`** — adds `withCredentials: true` so the browser includes HttpOnly cookies
2. **`loadingInterceptor`** — tracks in-flight requests for a global progress bar
3. **`authInterceptor`** — intercepts 401 responses, attempts a silent token refresh, and retries the failed request. If the refresh fails, marks the session as expired
4. **`errorInterceptor`** — handles all other HTTP errors (403, 0, 5xx) with user-facing notifications

---

## Functional Interceptors

Interceptors are plain functions — no class, no `implements HttpInterceptor`.

> **No `Authorization` header interceptor.** The JWT is stored in an `HttpOnly` cookie (`access_token`). The browser sends it automatically on every same-origin request — JavaScript cannot read it. `credentialsInterceptor` ensures `withCredentials: true` on every request. Do not write an interceptor that reads a token from a signal or sets an `Authorization` header.

### Auth Interceptor

Intercepts 401 responses on non-auth endpoints and attempts a silent refresh. Uses a `BehaviorSubject` to queue concurrent 401s — exactly one refresh call is made while all other failing requests wait and retry once the refresh completes.

```typescript
// core/interceptors/auth.interceptor.ts
import { HttpErrorResponse, HttpEvent, HttpHandlerFn, HttpInterceptorFn, HttpRequest } from '@angular/common/http';
import { inject } from '@angular/core';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { catchError, filter, switchMap, take } from 'rxjs/operators';
import { AuthApiService } from '../../features/auth/services/auth-api.service';
import { AuthService } from '../services/auth.service';

const AUTH_URLS = ['/auth/login', '/auth/register', '/auth/refresh', '/auth/logout'];

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // Skip auth endpoints to prevent refresh loops
  if (AUTH_URLS.some(url => req.url.includes(url))) {
    return next(req);
  }

  const authApi = inject(AuthApiService);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status !== 401) return throwError(() => error);
      // Attempt refresh, retry original request on success
      // On refresh failure: authService.markSessionExpired() triggers the overlay
      return handleUnauthorized(req, next, authApi, authService);
    })
  );
};
```

**Rules:**
- Auth endpoints (`/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/logout`) are excluded to prevent loops
- If the refresh itself fails, `authService.markSessionExpired()` is called — this triggers the session-expired overlay (not a redirect)
- The `BehaviorSubject` is reset after a failed refresh so that future login sessions can refresh normally

### Error Interceptor

Handles non-401 HTTP errors. The 401 case is handled by `authInterceptor` — the error interceptor does not redirect or show notifications for 401.

```typescript
// core/interceptors/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from '../services/notification.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const notify = inject(NotificationService);

  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      switch (err.status) {
        case 401:
          // Handled by authInterceptor — do not show notification or redirect
          break;
        case 403:
          notify.showError('You do not have permission to do that.');
          break;
        case 0:
          notify.showError('Network error. Please check your connection.');
          break;
        default:
          notify.showError(err.error?.message ?? 'An unexpected error occurred.');
      }
      return throwError(() => err);
    })
  );
};
```

### Loading Interceptor (Optional)

Tracks in-flight requests and exposes a loading signal for a global progress bar.

```typescript
// core/interceptors/loading.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs';
import { LoadingService } from '../services/loading.service';

export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(LoadingService);
  loading.increment();
  return next(req).pipe(finalize(() => loading.decrement()));
};
```

```typescript
// core/services/loading.service.ts
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private activeRequests = signal(0);
  readonly isLoading = computed(() => this.activeRequests() > 0);

  increment() { this.activeRequests.update(n => n + 1); }
  decrement() { this.activeRequests.update(n => Math.max(0, n - 1)); }
}
```

---

## API Service Pattern

API services are pure HTTP clients. They construct URLs, set headers, and return typed `Observable<T>`. No business logic.

### Structure

```typescript
// features/queue/services/queue-api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { QueueEntry, CreateQueueEntryRequest } from '../models/queue.models';

@Injectable({ providedIn: 'root' })
export class QueueApiService {
  private http = inject(HttpClient);
  private readonly BASE = '/api/queues';

  getQueueEntries(businessId: string): Observable<QueueEntry[]> {
    return this.http.get<QueueEntry[]>(`${this.BASE}/${businessId}/entries`);
  }

  getQueueEntry(entryId: string): Observable<QueueEntry> {
    return this.http.get<QueueEntry>(`${this.BASE}/entries/${entryId}`);
  }

  joinQueue(request: CreateQueueEntryRequest): Observable<QueueEntry> {
    return this.http.post<QueueEntry>(`${this.BASE}/${request.businessId}/join`, request);
  }

  updateEntryStatus(entryId: string, status: QueueEntryStatus): Observable<QueueEntry> {
    return this.http.patch<QueueEntry>(`${this.BASE}/entries/${entryId}`, { status });
  }

  callNext(businessId: string): Observable<QueueEntry> {
    return this.http.post<QueueEntry>(`${this.BASE}/${businessId}/call-next`, {});
  }

  getQueueHistory(businessId: string, date?: string): Observable<QueueEntry[]> {
    const params = date ? new HttpParams().set('date', date) : undefined;
    return this.http.get<QueueEntry[]>(`${this.BASE}/${businessId}/history`, { params });
  }
}
```

### Customer API (No Auth)

```typescript
// features/customer/services/customer-api.service.ts
@Injectable({ providedIn: 'root' })
export class CustomerApiService {
  private http = inject(HttpClient);

  // Public endpoint — accessed via WhatsApp link token
  getStatus(token: string): Observable<CustomerQueueStatus> {
    return this.http.get<CustomerQueueStatus>(`/api/customer/status/${token}`);
  }
}
```

---

## Request and Response Models

All API payloads use typed interfaces. Separate request models from response shapes.

```typescript
// queue.models.ts

// Response shape — what the API returns
export interface QueueEntry {
  id: string;
  ticketNumber: number;
  customerName: string;
  serviceType: string;
  status: QueueEntryStatus;
  position: number;
  estimatedWait: number;  // in minutes
  joinedAt: string;       // ISO 8601
  calledAt: string | null;
  servedAt: string | null;
}

// Request body — what we send to the API
export interface CreateQueueEntryRequest {
  businessId: string;
  customerPhone: string;
  serviceType: string;
}

export type QueueEntryStatus = 'waiting' | 'called' | 'served' | 'no-show';
```

---

## Environment Configuration

Base URLs are configured per environment.

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiBaseUrl: 'http://localhost:3000',
  wsBaseUrl:  'ws://localhost:3000',
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiBaseUrl: 'https://api.queueapp.com',
  wsBaseUrl:  'wss://api.queueapp.com',
};
```

```typescript
// In an API service:
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class QueueApiService {
  private readonly BASE = `${environment.apiBaseUrl}/queues`;
  ...
}
```

---

## Error Handling in Stores

API errors surface in the Signal Store via `tapResponse`. The store holds the error state and components display it.

```typescript
// In store withMethods:
loadQueue: rxMethod<string>(
  pipe(
    tap(() => patchState(store, { loading: true, error: null })),
    switchMap(id =>
      api.getQueueEntries(id).pipe(
        tapResponse({
          next:  entries => patchState(store, setAllEntities(entries), { loading: false }),
          error: (err: HttpErrorResponse) =>
            patchState(store, { loading: false, error: err.error?.message ?? 'Failed to load queue' }),
        })
      )
    )
  )
),
```

```html
<!-- In component template: -->
@if (store.error()) {
  <app-error-banner [message]="store.error()!" />
}
```

---

## Testing API Services

```typescript
import { TestBed } from '@angular/core/testing';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { provideHttpClient } from '@angular/common/http';
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

  afterEach(() => httpMock.verify());

  it('should fetch queue entries', () => {
    const mockEntries: QueueEntry[] = [
      { id: '1', ticketNumber: 1, status: 'waiting', position: 1 } as QueueEntry
    ];

    service.getQueueEntries('biz-1').subscribe(entries => {
      expect(entries).toEqual(mockEntries);
    });

    const req = httpMock.expectOne('/api/queues/biz-1/entries');
    expect(req.request.method).toBe('GET');
    req.flush(mockEntries);
  });

  it('should POST to join queue', () => {
    const request: CreateQueueEntryRequest = {
      businessId: 'biz-1',
      customerPhone: '+966501234567',
      serviceType: 'haircut',
    };

    service.joinQueue(request).subscribe();

    const req = httpMock.expectOne('/api/queues/biz-1/join');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(request);
    req.flush({ id: 'new-entry' });
  });
});
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| HTTP calls directly in a component | Move to API service → consumed by store via `rxMethod` |
| Business logic in API service | API services are pure HTTP clients — logic belongs in the store |
| Class-based interceptor (`implements HttpInterceptor`) | Use functional `HttpInterceptorFn` |
| Not handling `status: 0` (network failure) | Handle it in the error interceptor or store |
| Hardcoded API URLs in services | Use `environment.apiBaseUrl` |
| Missing typed request/response models | All request bodies and responses must be typed |
| Writing an interceptor that reads a token and sets `Authorization: Bearer` | JWT lives in an `HttpOnly` cookie — JS cannot read it. `credentialsInterceptor` + the browser handle cookie transport automatically |
| Redirecting to `/login` on 401 in the error interceptor | 401 is handled by `authInterceptor` (silent refresh). The error interceptor must not redirect or show a notification for 401 |
| Calling `/auth/refresh` through the auth interceptor | Auth URLs must be excluded from the auth interceptor to prevent infinite loops |
| Omitting `credentialsInterceptor` from the interceptor chain | Without `withCredentials: true`, the browser will not include HttpOnly cookies on requests |
