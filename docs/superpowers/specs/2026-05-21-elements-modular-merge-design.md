# Elements Modular Merge Design

## Context

`ELEMENT-SKILL.md` currently contains a standalone `@clink-ai/clink-elements` integration guide. The main `clink-integ-skills` routing only mentions embedded form integration as an optional standard integration frontend path, so users who choose Elements do not get a complete path through backend session creation, frontend SDK wiring, host UI events, webhook reconciliation, and validation artifacts.

The goal is to merge the Elements capability into `clink-integ-skills` without turning it into a separate top-level integration path. Elements is a frontend completion path under standard integration, not an independent payment architecture.

## Decision

Use a modular merge:

- keep `SKILL.md` short and route Elements requests through Standard Integration
- move detailed Elements guidance into a new reference module
- extend runtime routing and artifact generation so Elements-specific prompts produce Elements-specific implementation guidance
- add review checks and tests so future changes do not regress this path

Do not paste the full `ELEMENT-SKILL.md` into `SKILL.md`.

## Architecture

Add a new module:

- `references/elements-integration.md`

This module should describe the Elements frontend path inside Standard Integration. It should cover:

- prerequisites: merchant backend creates the checkout session first
- frontend initialization through `loadClinkElements`
- required frontend inputs: `publishKey`, `environment`, and `sessionId`
- safe key boundary: frontend uses publish key only; Secret Key stays server-side
- element lifecycle: `createElement('paymentMethod')`, optional `currencySelect`, `mount`, `unmount`, `destroy`
- host UI responsibilities: submit button, amount display, visibility handling, loading state, error display
- event mapping: `submit-enabled`, `submit-visible`, `amount-change`, `session-init-success`, `session-success`, `session-pending`, `promo-code-error`, and `error`
- customization: locale, theme, primary color, radius, currency selector behavior, built-in section visibility
- optional promotion-code UI through `promoCodeChange`
- runtime locale/theme changes through `setLocale` and `setTheme`
- recovery for session expired, completed, unsupported, load failure, and API validation errors
- reconciliation boundary: frontend success events are UX signals; backend webhook and server-side reconciliation remain authoritative

The standard integration module should point to this module from Step 4, where it already discusses merchant frontend payment entry.

## Routing

Update `SKILL.md` Standard Integration routing to explicitly recognize Elements terms:

- `@clink-ai/clink-elements`
- `clink-elements`
- `loadClinkElements`
- `createElement`
- `paymentMethod`
- `currencySelect`
- `mount`
- `unmount`
- `submit-enabled`
- `submit-visible`
- `amount-change`
- `session-success`
- `session-pending`
- `promoCodeChange`
- embedded checkout
- iframe payment component

When these signals appear, the skill should still route to Standard Integration, but it should read:

- `references/retrieval-protocol.md`
- `references/standard-integration.md`
- `references/elements-integration.md`

The route should still ask for product mode when checkout session design is needed. For frontend-only implementation requests, it should also ask for or infer the frontend framework when project context is missing.

## Runtime Artifacts

Extend `lib/skill-runtime.mjs` with Elements signal detection. For Standard Integration prompts that include Elements terms, add Elements-specific artifacts:

- `elements_frontend_checklist`: frontend setup, package/CDN choice, publish key, environment, session ID, mount containers, lifecycle cleanup
- `elements_event_mapping`: host UI mapping for submit state, button visibility, amount display, success, pending, promotion-code error, and generic error
- `elements_error_handling_checklist`: handling for API validation, expired session, completed session, unsupported session UI mode, and load failure
- `elements_host_ui_todo`: implementation tasks for React, Vue, or native JS host pages

When the prompt mentions promotion codes, add:

- `promotion_code_ui_contract`: states for collapsed, expanded, loading/error, applied, and clear actions using `promoCodeChange`

The normal Standard Integration artifacts remain present because Elements still depends on backend checkout session creation, webhook setup, merchant order mapping, and reconciliation.

## Review Checklist

Add Elements checks to `references/review-checklist.md` under Standard Integration:

- confirms the frontend framework or asks for it before framework-specific code
- confirms backend-created checkout session before frontend initialization
- keeps Secret Key and webhook signing key out of frontend code
- uses `publishKey`, `environment`, and `sessionId` for `loadClinkElements`
- creates `paymentMethod` before `currencySelect`
- does not create duplicate elements from one SDK instance
- calls `destroy()` during component teardown
- maps `submit-enabled` as "can submit", not "disabled"
- handles `submit-visible` so host buttons do not conflict with built-in third-party payment buttons
- treats `amount-change` as the source for displayed amount, product, promotion, and tax UI
- treats `session-success` and `session-pending` as frontend UX signals only
- keeps webhook-driven backend state as the authoritative payment confirmation
- handles known SDK errors and gives the user a retry or session recreation path

## README Updates

Update both `README.md` and `README-zh.md` so the public skill description explicitly includes Elements embedded checkout guidance. The README should state that the skill can help with:

- React, Vue, and native JS Elements integration
- SDK lifecycle and iframe mount/unmount behavior
- event-to-host-UI mapping
- promotion-code UI integration
- locale/theme customization
- webhook and reconciliation boundaries for embedded checkout

## Testing

Add runtime tests for these cases:

- a prompt mentioning `@clink-ai/clink-elements` routes to `merchant_standard_integration`
- a React `loadClinkElements` prompt emits Elements artifacts
- a prompt mentioning `promoCodeChange` emits `promotion_code_ui_contract`
- an Elements prompt still preserves core Standard Integration artifacts such as `integration_checklist`, `webhook_handler_checklist`, and `merchant_order_mapping`
- Elements prompts do not route to merchant agent integration unless agent-specific signals dominate

Add structure checks if the repository already enforces reference module listing:

- `references/elements-integration.md` is present
- `SKILL.md` module map includes it
- README module tables include it

## Non-Goals

This merge should not:

- make Elements a fifth top-level route
- remove hosted checkout or configured link guidance
- generate project-specific final frontend code without enough framework and codebase context
- weaken webhook, idempotency, reconciliation, or production validation requirements
- expose internal environment naming to merchant-facing output beyond existing developer-focused mappings where needed
- treat frontend success events as payment authority

## Implementation Order

1. Create `references/elements-integration.md` from the durable parts of `ELEMENT-SKILL.md`.
2. Update `SKILL.md` routing, working method, and module map.
3. Update `references/standard-integration.md` Step 4 to point to the Elements module.
4. Update `references/output-artifacts.md` with Elements-specific artifacts.
5. Update `references/review-checklist.md` with Elements checks.
6. Update `lib/skill-runtime.mjs` with Elements signals and artifacts.
7. Add runtime tests and any needed structure tests.
8. Update `README.md` and `README-zh.md`.

## Open Review Point

Before implementation, decide whether to keep `ELEMENT-SKILL.md` as a maintainer source file, rename it into `references/elements-integration.md`, or remove it after the module is created. The recommended default is to migrate its durable content into `references/elements-integration.md` and then remove or ignore the root-level standalone skill file so there is one canonical source.
