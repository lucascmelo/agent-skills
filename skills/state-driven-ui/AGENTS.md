---
name: react-state-driven-ui
description: Agent guidance for applying the react-state-driven-ui creation skill.
---

# State-Driven UI Architecture (React) - Agent Rules

This file is canonical.
Follow it when using this skill.
Do not guess around these rules.

## Re-sync requirement
On activation:
- Re-read these sections:
  - Construction workflow
  - Output contract
  - Failure conditions
- Use rules/ for detail when needed.
- Use references/ for examples when needed.

Before final output:
- Check failure conditions.
- If any failure condition applies, stop and redesign.

## Table of contents
1. Construction workflow (mandatory)
2. Output contract (mandatory)
3. Rules (compiled)
4. Failure conditions (mandatory)
5. Reference index

---

## 1. Construction workflow (mandatory)

Follow this order. Do not skip steps.
Do not write component code before steps 1 to 5 are complete.

1. Identify state categories and owners
   - Server cache state (React Query)
   - Client workflow state (reducer state)
   - URL state (validated boundary input)
   - Ephemeral UI state (local only)

2. Define state model
   - Define one phase field for process meaning.
   - Define draft overlays keyed by stable ids when needed.
   - Define error and request coordination fields when needed.

3. Define events
   - Define user intent events.
   - Define system outcome events emitted by effects.

4. Define transitions
   - Implement transitions as a pure reducer.
   - Map event -> next state.
   - Mark invalid transitions explicitly.

5. Define side effects
   - Bind side effects to events, not to components.
   - Use React Query mutations for server writes.
   - Use mutation callbacks to emit outcome events.

6. Define selectors
   - Compute derived values from source state.
   - Do not store derived values.

7. Define component boundaries
   - View components render only.
   - Containers connect selectors and dispatch intents.
   - Effects live outside view components.

If behavior cannot be expressed without business conditionals in JSX, stop and redesign.

---

## 2. Output contract (mandatory)

When using this skill, output in this order:

1. State model
   - Include ownership category for each field.

2. Events
   - List events grouped by:
     - user intent
     - system outcomes

3. Transition map
   - Provide reducer rules:
     - allowed event handling
     - phase changes
     - invalid transitions

4. Side effects plan
   - List each effect:
     - trigger event
     - React Query mutation used
     - callback -> emitted outcome events
     - stale response handling strategy

5. Selectors
   - List derived values and source fields.

6. Component boundaries
   - List container and view responsibilities.
   - Specify which layer dispatches which events.

7. Implementation steps
   - Sequence changes in construction workflow order.

8. Design rationale
   - Why this state model
   - Why these phases and events
   - Why this effects boundary (React Query callbacks)
   - Why not alternative patterns (short)
   - Trade-offs and what is intentionally skipped

Do not output component code before items 1 to 4 exist.

---

## 3. Rules (compiled)

Read the corresponding files in rules/ for the source.
See [references/rules-summary.md](references/rules-summary.md) for complete rule details.

### Construction and modeling rules

#### Rule: UI is a projection of state
Source: rules/flow-ui-projection.md
- Do not allow render timing to affect behavior.

#### Rule: Components emit intent only  
Source: rules/flow-intents-only.md
- View components must not encode business rules.

#### Rule: Events drive all workflow state changes
Source: rules/reducer-events-drive-state.md
- Every workflow state change must be caused by an event.

#### Rule: Use one phase field for process meaning
Source: rules/reducer-single-phase.md
- Represent process meaning with a single phase field.

#### Rule: Replace boolean soup with explicit phases
Source: rules/fsm-no-boolean-soup.md
- Do not encode phase using combinations like isDirty && !isSaving.

### Effects and async rules

#### Rule: Do not run effects in view components
Source: rules/effects-no-effects-in-components.md
- Side effects must run outside view components.

#### Rule: React Query mutation callbacks emit outcome events
Source: rules/query-mutation-callbacks.md
- Use mutations for writes.

#### Rule: Guard against stale async outcomes
Source: rules/query-stale-response-guard.md
- Use requestId to ignore out-of-order callbacks.

### Cross-cutting behavior rules

#### Rule: Navigation blocking requires explicit decision phase
Source: rules/routing-explicit-decision-state.md
- Model confirmation as a phase.

#### Rule: URL is a validated boundary input
Source: rules/url-validated-boundary.md
- Parse and validate URL state at the boundary.

### Correctness and maintainability rules

#### Rule: Derived state is selectors, not stored fields
Source: rules/derive-derived-state-as-selectors.md
- Do not store derived flags.

#### Rule: Stable identity is mandatory
Source: rules/identity-stable-ids-and-keys.md
- Use stable ids for keys.

#### Rule: No lifecycle-dependent behavior
Source: rules/determinism-no-lifecycle-dependence.md
- Re-render must not change behavior without an event.

### Documentation and reasoning rules

#### Rule: Document reasoning at boundaries
Source: rules/docs-rationale-at-boundaries.md
- Provide Design rationale section in agent output.

---

## 4. Failure conditions (mandatory)

Stop and redesign if any condition is true:

- There is no single phase field controlling process meaning.
- Phase is inferred from multiple boolean flags.
- Business rules exist in JSX or view component hooks.
- Side effects are triggered conditionally inside components.
- A re-render can change behavior without an event and transition.
- Navigation can occur without an explicit decision phase.
- Server cache is mirrored into workflow state instead of using an overlay.
- Derived values are stored instead of computed.

When refusing:
- Name the violated rule.
- Name the missing state, event, or transition.
- Propose the minimal redesign.

---

## 5. Reference index
- rules/: modular rule sources
- references/patterns.md
- references/examples/draft-save-flow.md
- references/examples/navigation-blocking-flow.md
- references/examples/async-validation-flow.md
- references/checklists/implementation-checklist.md
