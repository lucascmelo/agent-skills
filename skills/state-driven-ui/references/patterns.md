# State-Driven UI Patterns

This document describes implementation patterns for state-driven UI in React with TypeScript and React Query.

## Draft Overlay Pattern

### Concept
Create draft data as overlay keyed by stable IDs, not mirroring server cache in client state.

### Implementation
```typescript
// Server data (React Query cache)
const serverData = useQuery(['item', itemId], fetchItem)

// Draft overlay (client workflow state)
interface DraftOverlay {
  [itemId: string]: {
    data: Partial<ItemData>
    lastModified: number
    isDirty: boolean
  }
}

// Merge function for rendering
const mergedData = useMemo(() => {
  const draft = draftOverlay[itemId]
  return draft ? { ...serverData.data, ...draft.data } : serverData.data
}, [serverData.data, draftOverlay[itemId]])
```

### Benefits
- No cache invalidation complexity
- Clear separation of concerns
- Easy rollback (delete overlay entry)
- Stable IDs prevent reference issues

## Phase Enum Pattern

### Concept
Use explicit phase enums instead of boolean states to represent workflow states.

### Implementation
```typescript
enum WorkflowPhase {
  LOADING = 'loading',
  EDITING = 'editing',
  SAVING = 'saving',
  SAVED = 'saved',
  ERROR = 'error'
}

interface WorkflowState {
  phase: WorkflowPhase
  error?: string
  lastSaved?: Date
}
```

### Benefits
- Impossible to have contradictory states
- Clear transition paths
- Type-safe phase checking
- Easy to extend with new phases

## Transition Naming Conventions

### Convention
Use past tense for state transitions, imperative for actions.

### Implementation
```typescript
// Transitions (state changes)
const transitions = {
  startedEditing: (state: WorkflowState) => ({ ...state, phase: WorkflowPhase.EDITING }),
  savedDraft: (state: WorkflowState) => ({ ...state, phase: WorkflowPhase.SAVED }),
  encounteredError: (state: WorkflowState, error: string) => ({ 
    ...state, 
    phase: WorkflowPhase.ERROR, 
    error 
  })
}

// Actions (user triggers)
const actions = {
  startEditing: () => dispatch(transitions.startedEditing),
  saveDraft: (data: DraftData) => saveMutation.mutate(data),
  discardDraft: () => dispatch(transitions.discardedDraft)
}
```

### Benefits
- Clear separation between state changes and user actions
- Consistent naming across codebase
- Easy to trace state flow

## URL State Synchronization Pattern

### Concept
Treat URL state as serialized view of key workflow state, validated at boundaries.

### Implementation
```typescript
// URL schema
interface UrlState {
  phase?: WorkflowPhase
  itemId?: string
  tab?: string
}

// Validation at boundary
const validateUrlState = (raw: Partial<UrlState>): UrlState => {
  return {
    phase: Object.values(WorkflowPhase).includes(raw.phase as WorkflowPhase) 
      ? raw.phase as WorkflowPhase 
      : undefined,
    itemId: raw.itemId?.match(/^[a-zA-Z0-9-]+$/) ? raw.itemId : undefined,
    tab: raw.tab?.match(/^[a-z]+$/) ? raw.tab : undefined
  }
}

// Sync to URL
useEffect(() => {
  const urlState: UrlState = {
    phase: workflowState.phase === WorkflowPhase.EDITING ? WorkflowPhase.EDITING : undefined,
    itemId: workflowState.itemId
  }
  router.replace({ query: urlState })
}, [workflowState.phase, workflowState.itemId])
```

### Benefits
- Shareable URLs
- Browser navigation support
- Type safety at boundaries
- Minimal URL pollution

## Deterministic Rendering Rules

### Concept
Components render purely based on state, no lifecycle-driven behavior.

### Implementation
```typescript
// Correct: Pure rendering based on state
const EditButton = ({ phase, onStartEditing }: { 
  phase: WorkflowPhase
  onStartEditing: () => void 
}) => {
  if (phase === WorkflowPhase.EDITING) {
    return <SaveButton />
  }
  
  return <button onClick={onStartEditing}>Edit</button>
}

// Incorrect: Lifecycle-driven behavior
const EditButton = () => {
  const [isEditing, setIsEditing] = useState(false)
  
  useEffect(() => {
    // Creates implicit state dependencies
    if (someCondition) {
      setIsEditing(true)
    }
  }, [someCondition])
  
  // ...
}
```

### Benefits
- Predictable rendering
- Easy testing
- No implicit dependencies
- Clear data flow

## Mutation Callback Pattern

### Concept
Use React Query mutation callbacks to dispatch state transitions.

### Implementation
```typescript
const useSaveMutation = (dispatch: Dispatch) => {
  return useMutation({
    mutationFn: saveItem,
    onMutate: async (data) => {
      // Cancel in-flight queries
      await queryClient.cancelQueries(['item', data.id])
      
      // Optimistic update
      const previousData = queryClient.getQueryData(['item', data.id])
      queryClient.setQueryData(['item', data.id], data)
      
      // Transition to saving state
      dispatch(transitions.startedSaving)
      
      return { previousData }
    },
    onSuccess: () => {
      dispatch(transitions.savedDraft)
    },
    onError: (error, variables, context) => {
      // Rollback optimistic update
      if (context?.previousData) {
        queryClient.setQueryData(['item', variables.id], context.previousData)
      }
      
      dispatch(transitions.encounteredError(error.message))
    },
    onSettled: () => {
      // Invalidate to ensure fresh data
      queryClient.invalidateQueries(['item'])
    }
  })
}
```

### Benefits
- Clear separation of concerns
- Automatic error handling
- Optimistic updates with rollback
- Consistent state transitions
