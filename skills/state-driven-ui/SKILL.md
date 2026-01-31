---
name: state-driven-ui
description: Creation skill for building deterministic React UIs using explicit state models and React Query effects
license: MIT
metadata:
  author: Lucas Cavalcanti
  version: 0.1.0
---

# State-Driven UI

## When to use
- Multi-step forms or workflows with clear phases
- Draft/edit flows with save/discard semantics
- Features requiring navigation blocking or confirmation
- Async validation with optimistic UI updates
- Complex user interactions with multiple outcomes

## When not to use
- Simple display components without state transitions
- Purely presentational UI without side effects
- One-off interactions without workflow complexity

## Non-negotiable invariants
1. **State-first reasoning**: Define state model before any component code
2. **Single source of truth**: Server cache via React Query, no client mirroring
3. **Explicit transitions**: All state changes must go through defined transitions
4. **Render-only components**: Components cannot trigger state changes directly
5. **Deterministic rendering**: Same state = same UI, no lifecycle-driven behavior

## State taxonomy and ownership rules

### Server cache (React Query)
- **Owner**: React Query cache
- **Content**: Authoritative data from server
- **Rules**: Never mutate directly, use mutations only
- **Example**: `useQuery(['user', id], fetchUser)`

### Client workflow state
- **Owner**: Feature controller (custom hook)
- **Content**: Draft data, phase information, validation state
- **Rules**: Immutable updates only, transition via actions
- **Example**: `{ phase: 'editing', draft: { ... }, validation: { ... } }`

### URL state (schema validated)
- **Owner**: Router + validation layer
- **Content**: Serializable view of key workflow state
- **Rules**: Validate at boundary, keep minimal
- **Example**: `?phase=editing&id=123`

### Ephemeral UI state
- **Owner**: Individual components
- **Content**: Transient UI (focus, hover, dropdown open)
- **Rules**: Local to component, no cross-component effects
- **Example**: `{ isDropdownOpen: false, focusedField: null }`

## Mandatory reasoning order
1. **Identify all possible states** the user can be in
2. **Define state model** with clear ownership boundaries
3. **Map state transitions** with triggers and guards
4. **Plan effects** using React Query mutation callbacks
5. **Design selectors** for derived state
6. **Define component boundaries** based on state ownership
7. **Implement components** as pure render functions

## Required design artifacts
- **State model**: TypeScript interfaces with ownership annotations
- **Transition table**: All possible state changes with triggers
- **Effects plan**: React Query mutations with callback mappings
- **Selectors**: Pure functions for derived state
- **Component boundaries**: Clear separation by state ownership

## Effects model (React Query)

### Mutation placement
- **Location**: Feature controller layer, not components
- **Pattern**: `useMutation` in custom hook, exposed as actions
- **Example**: `const { saveDraft, discardDraft } = useWorkflowActions()`

### Callback dispatching
- **onMutate**: Optimistic updates + transition to pending state
- **onSuccess**: Transition to success state, invalidate queries
- **onError**: Transition to error state, rollback optimistic updates
- **onSettled**: Cleanup, reset loading states

### Cancellation handling
- Abort stale mutations on new transitions
- Use mutation keys for deduplication
- Handle race conditions in callback order

## Forbidden patterns
- Direct state mutation in components
- Server cache mirroring in client stores
- Boolean state without explicit phase modeling
- Lifecycle-driven side effects in components
- Implicit state dependencies between components
- Mixed concerns in single state objects

## Rejection conditions
Stop and redesign if you encounter:
- Multiple components sharing mutable state directly
- State transitions that are not explicitly defined
- Components that trigger their own state changes
- Server data duplicated in client state
- Validation logic scattered across components
- Navigation blocking without explicit decision state

## Required output format

When using this skill, structure your response as:

```markdown
## State Model
[TypeScript interfaces with ownership]

## Transition Table
[All state changes with triggers and guards]

## Effects Plan
[React Query mutations with callback mappings]

## Selectors
[Derived state functions]

## Component Boundaries
[State ownership mapping]

## Implementation Notes
[Critical implementation details]
```
