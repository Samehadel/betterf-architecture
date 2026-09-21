# Frontend Testing — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Test layers

- Test state and derived values through observable behavior with API dependencies isolated.
- Test API clients for request construction, response mapping, and failure propagation.
- Test components through public inputs, rendered output, and user interactions.
- Use end-to-end tests for approved critical journeys, not every individual implementation detail.

## Angular testing patterns

Import standalone components into `TestBed`. Set signal inputs with `fixture.componentRef.setInput()` and allow rendering to settle before assertions. Use `HttpTestingController` for HTTP client tests and verify outstanding requests are resolved.

Test initial, loading, success, empty, and error states. Include changing inputs and asynchronous ordering where they affect behavior. Match fixtures to actual transport models rather than hiding missing fields behind type assertions.

## Selectors

Prefer accessible roles, names, and labels for user interactions. Use stable test identifiers where semantic selectors are insufficient. Avoid CSS class and element-position dependencies that change with presentation refactors.

## End-to-end coverage

Derive journeys from BetterF requirements. Include relevant access restrictions, navigation, error recovery, keyboard use, and responsive layouts. Keep fixtures isolated and repeatable; avoid relying on uncontrolled shared data.

Use Playwright for browser tests. Configure browsers, server startup, and authentication fixtures when the application is initialized.

## Commands and coverage

Record verified commands from the selected build and CI. Coverage targets should reflect risk and meaningful behavior; no inherited percentage or test-runner configuration is an accepted BetterF gate.
