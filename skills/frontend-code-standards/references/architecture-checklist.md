# Architecture Checklist

Read this before restructuring a page or feature. Preserve the repository's established routing, state, data-fetching, component-library, and test conventions.

## Feature Boundary

Use this layout when it matches the existing repository. Do not create empty folders or abstractions for one-off code.

```text
features/<feature>/
  pages/             # Route-level composition and page-only layout
  components/        # Feature UI with an explicit public contract
  composables/       # Reusable stateful behavior (hooks in React)
  services/          # API calls, adapters, request/response mapping
  types/             # Domain contracts
  constants/         # Stable configuration and component registries
  __fixtures__/      # Mock responses and test data
```

## Ownership

| Concern | Owner | Avoid |
| --- | --- | --- |
| Route params, navigation, page layout, coordination | Page/container | Router access in presentational children |
| Repeated visual unit or self-contained dialog | Feature component | Copying its markup across a page |
| Transient form/input state | Closest interactive component | Global store for ephemeral child state |
| Shared feature behavior or async lifecycle | Composable/hook | Repeating page methods in components |
| HTTP and transport-to-domain mapping | Service/repository | `fetch` and response parsing in a component |
| Fixtures and fake responses | `__fixtures__/` plus mock adapter | Large literals in `data()`, `setup()`, `ref()`, or JSX/template |
| Status maps, options, step definitions | Module constant or `constants/` | Recreating configuration in component state |

Components receive domain data through typed props and emit user intent through typed events/callbacks. Do not mutate props or pass a whole store/page object merely to avoid defining a clear contract.

## Split Signals

Line counts trigger an architecture review; they are not automatic split limits. Split when a page mixes independent panels, dialogs, request lifecycles, or state families. Strong signals include:

- A `.vue`/`.tsx` file over 400 lines, template over 200 lines, or script over 250 lines, **and** a separable responsibility is present.
- A `v-if`/`v-else-if` workflow with three or more self-contained step views.
- Multiple dialogs in one file.
- A component that manages both a list and unrelated detail editing.
- Repeated markup, transformations, validation, or async state handling.

These are review signals, not permission to mechanically fragment every file. Keep a cohesive component intact when extraction would hide its meaning, add prop plumbing, or create a single-use wrapper. Prefer one readable 450-line feature component with one owner over several files whose boundaries are artificial.

## Reuse Decisions

- Extract a shared component/composable only when at least two meaningful consumers share invariant behavior and the same public contract.
- Keep variants local when their domain language, validation, lifecycle, or data shape differs.
- Prefer `OrderSummary` or `AddressForm` over a generic `BasePanel` with many mode flags and `unknown` data.
- Use configuration for genuinely identical repeated structure. Do not create a giant generic renderer that interprets business rules from strings.
- Keep cross-feature shared components small and stable; feature components remain with their feature by default.

## Multi-Step Flows

Replace a long Vue conditional chain with a step registry and a dynamic component. Keep step order, current index, navigation guards, and final submission in the parent. Each step owns only local layout, fields, and validation.

```ts
export const STEPS = [
  { id: 'basic', title: 'Basic information', component: StepBasicInfo },
  { id: 'team', title: 'Team details', component: StepTeamBuild },
] as const
```

```vue
<component
  :is="currentStep.component"
  :form-data="formData"
  :disabled="isViewMode"
  @update="handleStepUpdate"
  @next="goNext"
/>
```

Use a step-specific composable only when it makes validation or behavior reusable or materially easier to read. Keep derived display values in `computed`/`useMemo`, not in complex template expressions.

## Service and Mock Boundary

Define a small domain service interface. Production code implements it with the configured HTTP client; development and tests provide an implementation that imports fixtures. The page/container consumes the service through a composable or existing data layer.

```ts
export interface ProfileService {
  getProfile(): Promise<Profile>
  updateProfile(input: ProfileDraft): Promise<Profile>
}
```

This makes replacing mock data a one-line dependency change and prevents fixtures from becoming invisible component state.

## API Discovery and Placement

Before implementing an endpoint, search globally for the domain name, route fragment, HTTP method, API directory, request client, related types, and existing callers. The search establishes the project's real convention rather than guessing one.

Place the new call in the existing domain API/service module. Reuse the configured client, authentication, interceptors, response-envelope mapping, error policy, and request/response types. Components and pages call the feature's service/composable; they do not create parallel `fetch`/`axios` calls.

Create a new API module only when no domain module exists. Put it beside the closest comparable API module and follow its naming, exports, and test/mock conventions. Never add a catch-all API file, duplicate an endpoint that already exists, or relocate unrelated endpoints while implementing one feature.

## Refactoring Order

1. Record behavior and all reachable loading, empty, error, disabled, and success states.
2. Apply formatting and move constants, types, fixtures, and configuration without changing behavior.
3. Consolidate true duplication and fix direct-reference mutation bugs.
4. Move transport and request lifecycle into services/composables where a real boundary exists.
5. Extract dialogs, repeated units, and independent step views using explicit contracts.
6. Remove obsolete code, then run focused tests, lint, and type checks.
