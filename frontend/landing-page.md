# Public landing page — BTF-10

## Scope and authority

[BTF-10](https://linear.app/betterf/issue/BTF-10) selects the Orbit HTML/CSS mockup and its charcoal/lavender palette. The product owner confirmed “Explore the vision” links to the illustrative preview until enterprise registration is implemented under BTF-5. This is a public informational page, not an onboarding or product application.

## Implementation

- `/` lazy-loads a standalone, OnPush Angular landing page. Unknown paths redirect to `/`. No authentication guard or backend request is needed.
- Angular bootstrap uses zoneless change detection and the shared router. English copy is bundled through Transloco; locale selection and persistence are not introduced.
- The preview is semantic static HTML, explicitly labeled illustrative. Search, workspace navigation, and example knowledge entries do not act as live controls.
- Primary actions use native fragment links to `#workflow`; secondary navigation links to `#why`. A skip link targets the main content.
- Shared global styles retain the Orbit composition. Semantic CSS custom properties define background, surface, raised surface, border, primary/secondary text, accent, and accent text. These are the future theme boundary; only the selected medium-dark values ship, without a switch or preference storage.
- Tailwind is configured for shared utility styling. Responsive layouts preserve navigation and stack cards and the preview on small screens. Visible focus and reduced-motion handling support keyboard and motion-sensitive users.
- No API client, state store, form, analytics, remote font, registration endpoint, or user persistence is required for this static page.

## Validation

Run `npm ci`, `npm run typecheck`, and `npm run build` in `app/frontend`. Verify the public page, all fragment links, keyboard skip/focus behavior, and mobile/desktop layouts in a browser. Hosting must serve the Angular entry point for client-side routes.

The frontend bootstrap is included with BTF-10 because the application develop baseline (`4851bf4`) has no committed application runtime. Architecture baseline consulted: `ab3aa138af3f1b0d23d12661cc1c5849ab54a5ab`.
