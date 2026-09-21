# Routing — Implementation Guide

> Detailed implementation reference for Angular Router with standalone components, lazy loading, functional guards, and route-level providers.

---

## Setup in `app.config.ts`

```typescript
import { provideRouter, withComponentInputBinding, withViewTransitions } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),    // Route params/queryParams → input() signals
      withViewTransitions(),          // CSS View Transitions API on navigation
    ),
  ]
};
```

`withComponentInputBinding()` is especially important — it lets route parameters bind directly to `input()` signals on components, eliminating most `ActivatedRoute` usage.

---

## Route Definitions

### Root Routes (`app.routes.ts`)

```typescript
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full'
  },

  // Public routes — no guard
  {
    path: 'login',
    loadComponent: () => import('./features/auth/pages/login-page.component')
      .then(m => m.LoginPageComponent),
  },
  {
    // Customer status page — accessed via WhatsApp link, no auth required
    path: 'status/:token',
    loadComponent: () => import('./features/customer/pages/customer-status-page.component')
      .then(m => m.CustomerStatusPageComponent),
  },

  // Protected routes
  {
    path: 'dashboard',
    loadComponent: () => import('./features/dashboard/pages/dashboard-page.component')
      .then(m => m.DashboardPageComponent),
    canActivate: [authGuard],
  },
  {
    path: 'queue/:businessId',
    loadChildren: () => import('./features/queue/queue.routes')
      .then(m => m.queueRoutes),
    canActivate: [authGuard],
  },
  {
    path: 'business',
    loadChildren: () => import('./features/business/business.routes')
      .then(m => m.businessRoutes),
    canActivate: [authGuard],
  },

  // Fallback
  {
    path: '**',
    loadComponent: () => import('./shared/components/not-found/not-found.component')
      .then(m => m.NotFoundComponent),
  },
];
```

### Feature Routes (`queue.routes.ts`)

Feature route files define child routes and register feature-level providers (stores, services).

```typescript
import { Routes } from '@angular/router';
import { QueueStore } from './store/queue.store';

export const queueRoutes: Routes = [
  {
    path: '',
    providers: [QueueStore],           // Store is scoped to this route subtree
    children: [
      {
        path: '',
        loadComponent: () => import('./pages/queue-management-page.component')
          .then(m => m.QueueManagementPageComponent),
      },
      {
        path: 'settings',
        loadComponent: () => import('./pages/queue-settings-page.component')
          .then(m => m.QueueSettingsPageComponent),
      },
      {
        path: 'history',
        loadComponent: () => import('./pages/queue-history-page.component')
          .then(m => m.QueueHistoryPageComponent),
      },
    ]
  }
];
```

---

## Route Parameter Binding

With `withComponentInputBinding()`, route params, query params, and resolver data bind directly to `input()` signals — no `ActivatedRoute` injection needed.

```typescript
// Route: /status/:token
@Component({ standalone: true, ... })
export class CustomerStatusPageComponent implements OnInit {
  // Bound automatically from :token route param
  token = input.required<string>();

  ngOnInit() {
    this.statusService.connect(this.token());
  }
}

// Route: /queue/:businessId
@Component({ standalone: true, ... })
export class QueueManagementPageComponent implements OnInit {
  businessId = input.required<string>();

  ngOnInit() {
    this.store.loadQueue(this.businessId());
  }
}
```

For query parameters:
```typescript
// URL: /queue/biz-1?view=compact
view = input<string>('full');    // receives 'compact'
```

For resolver data:
```typescript
// In route: resolve: { business: businessResolver }
business = input.required<Business>();
```

---

## When to Use `loadComponent` vs `loadChildren`

| Use | When |
|---|---|
| `loadComponent` | A single page component — most routes |
| `loadChildren` | A feature with multiple sub-routes, or when you need route-level `providers` |

```typescript
// Single page — loadComponent
{
  path: 'dashboard',
  loadComponent: () => import('./features/dashboard/pages/dashboard-page.component')
    .then(m => m.DashboardPageComponent),
}

// Feature with sub-routes — loadChildren
{
  path: 'queue/:businessId',
  loadChildren: () => import('./features/queue/queue.routes')
    .then(m => m.queueRoutes),
}
```

---

## Route-Level Providers

Registering providers in the route `providers` array scopes them to that route subtree. This is how feature stores are provided — they only exist while the route is active.

```typescript
// queue.routes.ts
{
  path: '',
  providers: [
    QueueStore,                          // NgRx Signal Store
    QueueApiService,                     // Route-scoped feature dependency when needed
  ],
  children: [...]
}
```

This is equivalent to `providedIn: 'any'` but scoped per route — each route activation gets a fresh instance, and it's destroyed when the route deactivates.

---

## Functional Guards

Guards are plain functions returning `boolean`, `UrlTree`, or their Observable/Promise equivalents.

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth   = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) return true;

  // Preserve the intended URL for redirect after login
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};
```

```typescript
// role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const roleGuard = (requiredRole: string): CanActivateFn =>
  () => {
    const auth   = inject(AuthService);
    const router = inject(Router);

    return auth.hasRole(requiredRole) || router.createUrlTree(['/unauthorized']);
  };

// Usage:
canActivate: [roleGuard('admin')]
```

### Unsaved Changes Guard

```typescript
// unsaved-changes.guard.ts
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> =
  (component) => {
    if (!component.hasUnsavedChanges()) return true;
    return confirm('You have unsaved changes. Leave anyway?');
  };
```

---

## Programmatic Navigation

```typescript
@Component({ ... })
export class SomeComponent {
  private router = inject(Router);

  goToDashboard() {
    this.router.navigate(['/dashboard']);
  }

  goToQueue(businessId: string) {
    this.router.navigate(['/queue', businessId]);
  }

  goToStatus(token: string) {
    this.router.navigate(['/status', token]);
  }

  goBackAfterLogin(returnUrl: string) {
    this.router.navigateByUrl(returnUrl || '/dashboard');
  }
}
```

---

## Active Link Styling

```html
<!-- routerLinkActive applies class when route is active -->
<a
  routerLink="/dashboard"
  routerLinkActive="bg-indigo-50 text-indigo-700"
  ariaCurrentWhenActive="page"
  class="flex items-center gap-2 rounded-md px-3 py-2 text-sm font-medium text-gray-600"
>
  Dashboard
</a>
```

Import `RouterLink` and `RouterLinkActive` in the component's `imports` array.

---

## Resolvers

Resolvers pre-fetch data before a route activates. Use functional resolvers.

```typescript
// business.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { BusinessApiService } from '../services/business-api.service';

export const businessResolver: ResolveFn<Business> =
  (route) => inject(BusinessApiService).getBusiness(route.paramMap.get('businessId')!);

// In route:
{
  path: ':businessId',
  resolve: { business: businessResolver },
  loadComponent: () => import('./pages/business-detail-page.component')
    .then(m => m.BusinessDetailPageComponent),
}

// In component — bound automatically via withComponentInputBinding():
business = input.required<Business>();
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `AppRoutingModule` | Delete it — use `provideRouter(routes)` in `app.config.ts` |
| `ActivatedRoute.params` subscription | Use `input()` via `withComponentInputBinding()` |
| Class-based guards (`CanActivate` interface) | Use functional `CanActivateFn` |
| Providing feature stores in `app.config.ts` | Provide them in the route's `providers` array |
| `RouterModule.forRoot()` / `forChild()` | Use `provideRouter()` and route files with exported `Routes` arrays |
