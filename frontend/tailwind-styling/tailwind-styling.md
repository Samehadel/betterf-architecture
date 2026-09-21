# Tailwind CSS Styling — Implementation Guide

> Detailed reference for styling conventions in the WhatsApp Virtual Queue frontend. All styling uses Tailwind CSS utility classes directly in templates.

---

## Setup

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

export default {
  content: ['./src/**/*.{html,ts}'],
  theme: {
    extend: {
      colors: {
        brand: {
          50:  '#eef2ff',
          500: '#6366f1',   // Indigo — primary brand color
          600: '#4f46e5',
          700: '#4338ca',
        }
      }
    }
  },
  plugins: [],
} satisfies Config;
```

```css
/* src/styles.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

No per-component CSS files. No CSS Modules. No `styleUrls`. All styling lives in templates.

---

## Status Badge Pattern

Queue entries have statuses that need consistent visual treatment across all components.

```html
<!-- Inline conditional class binding -->
<span
  class="rounded-full px-3 py-1 text-xs font-medium"
  [class]="entry().status === 'waiting' ? 'bg-yellow-100 text-yellow-800'
         : entry().status === 'called'  ? 'bg-blue-100 text-blue-800'
         : entry().status === 'served'  ? 'bg-green-100 text-green-800'
                                        : 'bg-gray-100 text-gray-600'"
>
  {{ entry().status | titlecase }}
</span>
```

For more complex conditional class logic, compute a class string in the component:

```typescript
statusClass = computed(() => {
  const map: Record<QueueEntryStatus, string> = {
    waiting: 'bg-yellow-100 text-yellow-800',
    called:  'bg-blue-100 text-blue-800',
    served:  'bg-green-100 text-green-800',
    'no-show': 'bg-gray-100 text-gray-600',
  };
  return map[this.entry().status];
});
```

```html
<span class="rounded-full px-3 py-1 text-xs font-medium" [class]="statusClass()">
  {{ entry().status | titlecase }}
</span>
```

---

## Layout Patterns

### Page Shell

```html
<div class="min-h-screen bg-gray-50">

  <!-- Top nav -->
  <header class="border-b border-gray-200 bg-white px-6 py-4">
    <div class="mx-auto flex max-w-6xl items-center justify-between">
      <h1 class="text-lg font-semibold text-gray-900">Queue Dashboard</h1>
      <nav class="flex gap-4">...</nav>
    </div>
  </header>

  <!-- Main content -->
  <main class="mx-auto max-w-6xl px-4 py-8">
    ...
  </main>

</div>
```

### Card

```html
<div class="rounded-xl border border-gray-200 bg-white p-6 shadow-sm">
  ...
</div>
```

### Queue Ticket Row

```html
<div class="flex items-center gap-4 rounded-lg border border-gray-100 bg-white p-4 shadow-sm transition hover:shadow-md">

  <!-- Position number -->
  <div class="flex h-10 w-10 shrink-0 items-center justify-center rounded-full bg-indigo-50 text-sm font-bold text-indigo-700">
    {{ entry().position }}
  </div>

  <!-- Customer info -->
  <div class="flex-1 min-w-0">
    <p class="truncate font-medium text-gray-900">{{ entry().customerName }}</p>
    <p class="text-sm text-gray-500">{{ entry().serviceType }}</p>
  </div>

  <!-- Wait time -->
  <p class="text-sm text-gray-500 shrink-0">~{{ entry().estimatedWait }} min</p>

  <!-- Actions -->
  <div class="flex shrink-0 gap-2">
    <button
      (click)="called.emit(entry().id)"
      class="rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700 transition-colors"
    >
      Call
    </button>
    <button
      (click)="noShow.emit(entry().id)"
      class="rounded-md border border-gray-300 px-3 py-1.5 text-sm text-gray-600 hover:bg-gray-50 transition-colors"
    >
      No Show
    </button>
  </div>

</div>
```

### Stats Bar

```html
<div class="grid grid-cols-2 gap-4 sm:grid-cols-4">
  <div class="rounded-lg bg-white p-4 shadow-sm text-center">
    <p class="text-3xl font-bold text-indigo-600">{{ waitingCount() }}</p>
    <p class="mt-1 text-sm text-gray-500">Waiting</p>
  </div>
  <!-- repeat for other stats -->
</div>
```

---

## Mobile-First — Customer Status Page

The customer status page is accessed from WhatsApp on a phone. Design mobile-first, then enhance for larger screens.

```html
<!-- Customer status — phone-optimized -->
<div class="min-h-screen bg-indigo-600 flex items-center justify-center px-4">
  <div class="w-full max-w-sm rounded-2xl bg-white p-8 shadow-xl text-center">

    <p class="text-sm font-medium uppercase tracking-wide text-indigo-600">Your Queue Position</p>

    <p class="mt-4 text-7xl font-black text-gray-900">{{ status()?.position }}</p>

    <p class="mt-2 text-gray-500">
      Estimated wait: <span class="font-semibold text-gray-700">~{{ status()?.estimatedWait }} min</span>
    </p>

    <div class="mt-6 rounded-lg bg-indigo-50 p-4">
      <p class="text-sm text-indigo-800">We'll send you a WhatsApp message when it's your turn.</p>
    </div>

  </div>
</div>
```

---

## Responsive Prefixes

Use `sm:`, `md:`, `lg:`, `xl:` for progressive enhancement. Always start with the mobile layout.

```html
<!-- 1 column on mobile, 2 on tablet, 4 on desktop -->
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  ...
</div>

<!-- Stack on mobile, side-by-side on tablet+ -->
<div class="flex flex-col gap-4 md:flex-row">
  ...
</div>

<!-- Hide on mobile, show on desktop -->
<aside class="hidden lg:block">...</aside>
```

---

## Dark Mode (if needed)

Enable in `tailwind.config.ts`:

```typescript
export default {
  darkMode: 'class',    // Toggle by adding 'dark' class to <html>
  ...
}
```

Then use `dark:` variants:

```html
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
  ...
</div>
```

---

## `@apply` — When to Use

`@apply` is acceptable **only** for repeated utility groups that are shared across multiple unrelated templates and would otherwise require duplication. Use it sparingly — in `styles.css` only.

```css
/* styles.css — acceptable use */
@layer components {
  .btn-primary {
    @apply rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white
           hover:bg-indigo-700 transition-colors focus:outline-none focus:ring-2
           focus:ring-indigo-500 focus:ring-offset-2;
  }

  .card {
    @apply rounded-xl border border-gray-200 bg-white p-6 shadow-sm;
  }
}
```

Do not create per-component CSS files using `@apply` — put it all in `styles.css`.

---

## Common Class Groups Reference

| Purpose | Classes |
|---|---|
| Card container | `rounded-xl border border-gray-200 bg-white p-6 shadow-sm` |
| Primary button | `rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 transition-colors` |
| Secondary button | `rounded-md border border-gray-300 px-4 py-2 text-sm text-gray-600 hover:bg-gray-50 transition-colors` |
| Danger button | `rounded-md bg-red-600 px-4 py-2 text-sm font-medium text-white hover:bg-red-700 transition-colors` |
| Input field | `block w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500` |
| Section heading | `text-lg font-semibold text-gray-900` |
| Muted text | `text-sm text-gray-500` |
| Page container | `mx-auto max-w-6xl px-4 py-8` |
| Status: waiting | `bg-yellow-100 text-yellow-800` |
| Status: called | `bg-blue-100 text-blue-800` |
| Status: served | `bg-green-100 text-green-800` |
| Status: no-show | `bg-gray-100 text-gray-600` |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Per-component `.css` files with custom styles | Delete them — use Tailwind utilities in templates |
| Using `[ngClass]` with object syntax | Use `[class]` with computed signal string |
| Hardcoded inline `style=""` attributes | Replace with Tailwind classes |
| Mobile layout built last | Always start with mobile, add `sm:`/`md:`/`lg:` to enhance |
| Over-using `@apply` | Reserve it for globally shared class groups in `styles.css` |
