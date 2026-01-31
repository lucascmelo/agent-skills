---
name: state-driven-ui-agents
description: Agent behavior contract for state-driven-ui creation skill
license: MIT
metadata:
  version: 0.1.0
---

# State-Driven UI Agents

## Purpose
This skill enables agents to create deterministic React UIs by enforcing state-first reasoning and explicit transition modeling. Agents must follow strict reasoning order and produce structured outputs.

## Activation keywords and triggers
Use this skill when you encounter:
- "editable flow" or "draft functionality"
- "save/discard" semantics
- "navigation confirm" or "blocking"
- "multi-step" or "wizard"
- "async validation" with optimistic updates
- Complex form workflows with multiple outcomes

## Agent behavior contract

### Mandatory pre-conditions
Before generating component code, the agent MUST:
1. Define complete state model with ownership annotations
2. Map all transitions with triggers and guards
3. Plan React Query effects with callback mappings
4. Design selectors for derived state
5. Establish component boundaries based on state ownership

### Reasoning enforcement
- Never skip steps: Follow mandatory reasoning order exactly
- State-first: No component code until state model complete
- Explicit transitions: Every state change must be defined
- Ownership clarity: Each piece of state has single owner

### Rejection conditions
The agent MUST refuse solutions that:
- Violate forbidden patterns in SKILL.md
- Skip mandatory reasoning order
- Generate components before state design
- Mix concerns across state ownership boundaries
- Use implicit state dependencies

### Output compliance
Every response using this skill MUST follow the required output format exactly, with all sections completed.

## Quality gates
The agent must self-validate:
- Are all invariants from SKILL.md satisfied?
- Is state ownership clearly defined?
- Are transitions explicit and comprehensive?
- Are effects properly mapped to React Query callbacks?
- Are components render-only with clear boundaries?

## Error handling
If rejection conditions encountered, the agent must:
1. Stop immediately
2. Identify which invariant or rule is violated
3. Explain why current approach is invalid
4. Propose redesigned approach that satisfies all requirements
