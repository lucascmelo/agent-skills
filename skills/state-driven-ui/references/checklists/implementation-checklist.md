# Implementation Checklist

Use this checklist to validate state-driven-ui implementations before coding and before merging.

## Pre-Implementation Checklist

### State Modeling
- [ ] All possible states identified
- [ ] State ownership defined
- [ ] State boundaries established
- [ ] State model typed
- [ ] Initial state defined

### Transition Planning
- [ ] All transitions mapped
- [ ] Transition triggers identified
- [ ] Transition guards defined
- [ ] Transition naming consistent
- [ ] No unreachable states

### Effects Planning
- [ ] React Query mutations planned
- [ ] Callback mappings defined
- [ ] Optimistic updates planned
- [ ] Error handling planned
- [ ] Cancellation strategy

### Component Design
- [ ] Component boundaries clear
- [ ] Props interfaces defined
- [ ] Render-only components
- [ ] Selector functions planned
- [ ] No implicit dependencies

## Implementation Checklist

### State Implementation
- [ ] State immutable
- [ ] State ownership respected
- [ ] Type safety enforced
- [ ] Default values safe
- [ ] State serialization

### Transition Implementation
- [ ] All transitions implemented
- [ ] Transition guards enforced
- [ ] Transition logging
- [ ] Transition atomicity
- [ ] No direct state mutation

### Effects Implementation
- [ ] Mutations in controller layer
- [ ] Callback dispatching correct
- [ ] Optimistic updates with rollback
- [ ] Query invalidation correct
- [ ] Loading states managed

### Component Implementation
- [ ] Components render-only
- [ ] Props flow correct
- [ ] Ephemeral state local
- [ ] No lifecycle side effects
- [ ] Deterministic rendering

### Integration Points
- [ ] Router integration
- [ ] URL synchronization
- [ ] React Query integration
- [ ] Error boundaries
- [ ] Performance considerations

## Pre-Merge Checklist

### Code Quality
- [ ] No forbidden patterns
- [ ] Consistent naming
- [ ] TypeScript strict mode
- [ ] No console.log
- [ ] Code organization

### Testing
- [ ] State transitions tested
- [ ] Error cases tested
- [ ] Edge cases tested
- [ ] Integration tested
- [ ] Performance tested

### Documentation
- [ ] State model documented
- [ ] Transitions documented
- [ ] Component contracts documented
- [ ] Usage examples provided
- [ ] README updated

### Review Requirements
- [ ] Skill invariants satisfied
- [ ] No rejection conditions
- [ ] Required output format
- [ ] Peer review completed
- [ ] Skill compliance verified

## Automated Validation

### CI Checks
- [ ] TypeScript compilation
- [ ] Linting passes
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Performance tests pass

### Runtime Validation
- [ ] State validation
- [ ] Transition validation
- [ ] Error monitoring
- [ ] Performance monitoring
- [ ] User experience metrics

## Final Sign-off

Before marking implementation complete:

1. Self-review completed
2. Skill compliance verified
3. Documentation complete
4. Testing adequate
5. Ready for production

## Common Issues to Watch For

### State Management
- Mixing concerns across state ownership boundaries
- Direct mutation of React Query cache
- Implicit state dependencies between components
- Missing transition definitions

### Effects Handling
- Mutations in components instead of controller layer
- Missing error handling in mutation callbacks
- Race conditions not handled
- No rollback strategy for optimistic updates

### Component Design
- Components triggering their own state changes
- Lifecycle-driven side effects
- Props drilling instead of proper state ownership
- Ephemeral state in wrong location

### Integration
- Navigation blocking without explicit decision state
- URL state not validated at boundaries
- Missing error boundaries
- Performance issues from unnecessary re-renders
