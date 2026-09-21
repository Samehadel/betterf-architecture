# Tailwind Styling — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Scope

Use Tailwind utility classes to express consistent spacing, layout, responsive behavior, and interaction states. Configure setup against the selected major version. Product colors, typography, and status styling remain project decisions. Use utilities in templates and keep shared CSS in the global stylesheet.

Use reusable components or shared style abstractions when repeated utility groups justify them. Keep keyboard focus visible and do not communicate meaning through color alone.

## Layout example

```html
<main class="mx-auto max-w-6xl px-4 py-8">
  <section class="rounded-xl border border-gray-200 bg-white p-6 shadow-sm">
    <!-- Feature content -->
  </section>
</main>
```

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

The class groups above are illustrative visual samples, not BetterF brand tokens. Verify contrast and interaction states in the final design. Add dark-mode and bidirectional variants only according to the chosen design requirements and library version.
