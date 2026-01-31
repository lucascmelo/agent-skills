# Draft Save Flow Example

## Problem statement
User needs to edit existing item with save/discard semantics, including optimistic updates and navigation blocking when unsaved changes exist.

## State model

### Server cache (React Query)
```typescript
interface ItemData {
  id: string
  title: string
  description: string
  updatedAt: Date
}

// Owner: React Query cache
const useItemQuery = (id: string) => 
  useQuery(['item', id], () => fetchItem(id))
```

### Client workflow state
```typescript
enum DraftPhase {
  VIEWING = 'viewing',
  EDITING = 'editing',
  SAVING = 'saving',
  SAVED = 'saved',
  ERROR = 'error'
}

interface DraftOverlay {
  [itemId: string]: {
    data: Partial<ItemData>
    isDirty: boolean
    lastModified: number
  }
}

interface WorkflowState {
  phase: DraftPhase
  activeItemId?: string
  drafts: DraftOverlay
  error?: string
  lastSaved?: Date
}

// Owner: Feature controller (useDraftWorkflow hook)
```

### Ephemeral UI state
```typescript
interface UIState {
  focusedField?: string
  isConfirmDialogOpen: boolean
  pendingNavigation?: string
}

// Owner: Individual components
```

## Transition table

| Current State | Trigger | Next State | Action |
|---------------|---------|------------|--------|
| VIEWING | startEditing(itemId) | EDITING | Create draft overlay |
| EDITING | updateField(field, value) | EDITING | Update draft overlay |
| EDITING | saveDraft() | SAVING | Trigger save mutation |
| EDITING | discardDraft() | VIEWING | Remove draft overlay |
| EDITING | attemptNavigation() | EDITING | Show confirm dialog |
| SAVING | onSuccess | SAVED | Clear draft, update cache |
| SAVING | onError | ERROR | Show error, keep draft |
| ERROR | retrySave() | SAVING | Retry save mutation |
| ERROR | discardDraft() | VIEWING | Remove draft overlay |
| SAVED | auto | VIEWING | Clear active item |

## Effects plan (React Query)

### Save mutation
```typescript
const useSaveMutation = (dispatch: Dispatch) => {
  return useMutation({
    mutationFn: ({ itemId, data }: { itemId: string, data: ItemData }) => 
      saveItem(itemId, data),
    
    onMutate: async ({ itemId, data }) => {
      // Cancel in-flight queries
      await queryClient.cancelQueries(['item', itemId])
      
      // Optimistic update
      const previousData = queryClient.getQueryData<ItemData>(['item', itemId])
      queryClient.setQueryData(['item', itemId], data)
      
      // Transition to saving
      dispatch({ type: 'STARTED_SAVING' })
      
      return { previousData, itemId }
    },
    
    onSuccess: (result, variables, context) => {
      dispatch({ type: 'SAVED_DRAFT', payload: { timestamp: new Date() } })
    },
    
    onError: (error, variables, context) => {
      // Rollback optimistic update
      if (context?.previousData) {
        queryClient.setQueryData(['item', context.itemId], context.previousData)
      }
      
      dispatch({ 
        type: 'ENCOUNTERED_ERROR', 
        payload: { error: error.message } 
      })
    },
    
    onSettled: (result, error, variables) => {
      // Invalidate to ensure fresh data
      queryClient.invalidateQueries(['item', variables.itemId])
    }
  })
}
```

## Selectors

```typescript
const selectors = {
  // Merged data for rendering
  currentItemData: (state: WorkflowState, serverData?: ItemData) => {
    if (!state.activeItemId || !serverData) return undefined
    
    const draft = state.drafts[state.activeItemId]
    return draft ? { ...serverData, ...draft.data } : serverData
  },
  
  // Is current item dirty?
  isDirty: (state: WorkflowState) => {
    if (!state.activeItemId) return false
    return state.drafts[state.activeItemId]?.isDirty ?? false
  },
  
  // Can navigate away?
  canNavigate: (state: WorkflowState) => {
    return state.phase === DraftPhase.VIEWING || state.phase === DraftPhase.SAVED
  },
  
  // Error message for display
  errorMessage: (state: WorkflowState) => {
    return state.phase === DraftPhase.ERROR ? state.error : undefined
  }
}
```

## Component boundaries

### ItemViewComponent
- **Owns**: Ephemeral UI state (focus, dialog state)
- **Receives**: Current item data, workflow state, actions
- **Renders**: Form fields, save/discard buttons
- **Cannot**: Directly mutate workflow state

### NavigationBlocker
- **Owns**: Navigation confirmation logic
- **Receives**: Can navigate flag, pending navigation
- **Renders**: Confirmation dialog
- **Cannot**: Access draft data directly

### WorkflowController (useDraftWorkflow)
- **Owns**: Complete workflow state, draft overlays
- **Receives**: Server data via React Query
- **Exposes**: Actions, selectors, phase
- **Cannot**: Render UI directly

## Implementation notes

### Draft overlay management
```typescript
// Update draft overlay
const updateDraft = (itemId: string, updates: Partial<ItemData>) => {
  dispatch({
    type: 'UPDATED_DRAFT',
    payload: {
      itemId,
      data: updates,
      isDirty: true,
      lastModified: Date.now()
    }
  })
}

// Discard draft
const discardDraft = (itemId: string) => {
  dispatch({
    type: 'DISCARDED_DRAFT',
    payload: { itemId }
  })
}
```

### Navigation blocking
```typescript
// Block navigation when dirty
useBlocker(
  ({ currentLocation, nextLocation }) => {
    return !selectors.canNavigate(workflowState)
  },
  when(workflowState.activeItemId && selectors.isDirty(workflowState))
)
```

### Auto-save considerations
- Implement debounced auto-save in addition to manual save
- Use different mutation keys for auto vs manual saves
- Show clear UI feedback for auto-save status

## Edge cases

1. **Concurrent edits**: Abort previous save mutations when new ones start
2. **Network failures**: Keep draft intact, show retry option
3. **Server validation errors**: Display field-specific errors in draft
4. **Browser refresh**: Persist draft to sessionStorage for recovery
5. **Multiple items**: Support multiple simultaneous drafts with different IDs
