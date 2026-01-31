# State-Driven UI References

This directory contains implementation examples, patterns, and guidance for the state-driven-ui skill.

## Structure

- [`patterns.md`](patterns.md) - Implementation patterns
- [`examples/`](examples/) - Flow examples
  - [`draft-save-flow.md`](examples/draft-save-flow.md) - Draft editing with save/discard
  - [`navigation-blocking-flow.md`](examples/navigation-blocking-flow.md) - Navigation with confirmation
  - [`async-validation-flow.md`](examples/async-validation-flow.md) - Async validation with debouncing
- [`checklists/`](checklists/) - Implementation validation
  - [`implementation-checklist.md`](checklists/implementation-checklist.md) - Pre-implementation and pre-merge checklist

## Usage

References provide progressive disclosure - start with main skill documentation in [`../SKILL.md`](../SKILL.md), then review specific patterns and examples.

## Pattern Categories

### State Modeling
- Draft overlays keyed by stable IDs
- Phase enums vs state machines
- Transition naming conventions

### Effects Integration
- React Query mutation callbacks
- Optimistic updates and rollback
- Cancellation and race condition handling

### Implementation Patterns
- Component boundary definition
- Selector design
- URL state synchronization

See [`patterns.md`](patterns.md) for detailed explanations.
