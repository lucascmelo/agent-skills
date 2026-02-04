# Rules Summary

Complete rule details and implementation guidance for react-state-driven-ui skill.

## Construction and modeling rules

### UI is a projection of state
Source: rules/flow-ui-projection.md

- Do not allow render timing to affect behavior.
- Do not trigger transitions from rendering.
- Do not store meaning in component lifecycle.

### Components emit intent only
Source: rules/flow-intents-only.md

- View components must not encode business rules.
- View components must not orchestrate workflows or side effects.
- View components may render selector output and emit intent callbacks.

### Events drive all workflow state changes
Source: rules/reducer-events-drive-state.md

- Every workflow state change must be caused by an event.
- Reducer is the only place that updates workflow state.
- Reducer is pure and deterministic.

### Use one phase field for process meaning
Source: rules/reducer-single-phase.md

- Represent process meaning with a single phase field.
- Do not infer phase from multiple flags.
- Make illegal states unrepresentable.

### Replace boolean soup with explicit phases
Source: rules/fsm-no-boolean-soup.md

- Do not encode phase using combinations like isDirty && !isSaving.
- Use explicit phases and explicit transitions.
- Mark invalid transitions.

## Effects and async rules

### Do not run effects in view components
Source: rules/effects-no-effects-in-components.md

- Side effects must run outside view components.
- Do not coordinate workflows in useEffect.
- Effects must emit outcome events, then reducer updates state.

### React Query mutation callbacks emit outcome events
Source: rules/query-mutation-callbacks.md

- Use mutations for writes.
- Emit outcome events from callbacks.
- Do not mutate workflow state directly inside callbacks.

### Guard against stale async outcomes
Source: rules/query-stale-response-guard.md

- Use requestId to ignore out-of-order callbacks.
- Abort where possible, still guard with requestId.

## Cross-cutting behavior rules

### Navigation blocking requires explicit decision phase
Source: rules/routing-explicit-decision-state.md

- Model confirmation as a phase.
- Store pending destination in state.
- Navigate only after explicit confirm event.

### URL is a validated boundary input
Source: rules/url-validated-boundary.md

- Parse and validate URL state at the boundary.
- Store validated output in workflow state.
- Version schema when evolving.

## Correctness and maintainability rules

### Derived state is selectors, not stored fields
Source: rules/derive-derived-state-as-selectors.md

- Do not store derived flags.
- Compute derived values from source state.
- Memoize selectors, not component logic.

### Stable identity is mandatory
Source: rules/identity-stable-ids-and-keys.md

- Use stable ids for keys.
- Do not derive keys from mutable values.
- Avoid remount cascades.

### No lifecycle-dependent behavior
Source: rules/determinism-no-lifecycle-dependence.md

- Re-render must not change behavior without an event.
- Do not rely on mount/unmount to define meaning.
- Time affects scheduling, not correctness.

## Documentation and reasoning rules

### Document reasoning at boundaries
Source: rules/docs-rationale-at-boundaries.md

- Provide Design rationale section in agent output.
- Add short boundary comments only where needed.
- Do not comment every decision in code.
- Keep comments under 3 to 6 lines each.
