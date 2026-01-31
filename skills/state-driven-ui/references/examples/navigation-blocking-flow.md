# Navigation Blocking Flow Example

## Problem statement
User is in multi-step form or editing flow and attempts to navigate away. System must block navigation and explicitly ask for confirmation, with clear decision states.

## State model

### Server cache (React Query)
```typescript
interface FormData {
  id: string
  step: number
  data: Record<string, any>
  isComplete: boolean
}

// Owner: React Query cache
const useFormQuery = (id: string) => 
  useQuery(['form', id], () => fetchForm(id))
```

### Client workflow state
```typescript
enum NavigationPhase {
  NAVIGATING = 'navigating',
  BLOCKED = 'blocked',
  CONFIRMING = 'confirming',
  ALLOWED = 'allowed'
}

enum WorkflowPhase {
  FILLING = 'filling',
  DIRTY = 'dirty',
  SAVING = 'saving',
  SAVED = 'saved'
}

interface NavigationState {
  phase: NavigationPhase
  pendingNavigation?: {
    to: string
    trigger: 'browser' | 'link' | 'programmatic'
  }
  decision?: 'stay' | 'leave' | 'save_and_leave'
}

interface WorkflowState {
  phase: WorkflowPhase
  currentStep: number
  totalSteps: number
  hasUnsavedChanges: boolean
  navigation: NavigationState
}

// Owner: Feature controller (useNavigationWorkflow hook)
```

### Ephemeral UI state
```typescript
interface DialogState {
  isOpen: boolean
  title: string
  message: string
  options: Array<{ label: string; action: string; variant: 'primary' | 'secondary' }>
}

// Owner: ConfirmationDialog component
```

## Transition table

### Navigation transitions
| Current State | Trigger | Next State | Action |
|---------------|---------|------------|--------|
| NAVIGATING | hasUnsavedChanges | BLOCKED | Store pending navigation |
| BLOCKED | showConfirmation() | CONFIRMING | Show dialog |
| CONFIRMING | userChooses('stay') | ALLOWED | Clear pending navigation |
| CONFIRMING | userChooses('leave') | ALLOWED | Proceed with navigation |
| CONFIRMING | userChooses('save_and_leave') | SAVING | Trigger save then navigate |
| ALLOWED | auto | NAVIGATING | Reset navigation state |

### Workflow transitions
| Current State | Trigger | Next State | Action |
|---------------|---------|------------|--------|
| FILLING | fieldChanged() | DIRTY | Set unsaved flag |
| DIRTY | fieldChanged() | DIRTY | Update draft data |
| DIRTY | saveForm() | SAVING | Trigger save mutation |
| SAVING | onSuccess | SAVED | Clear unsaved flag |
| SAVING | onError | DIRTY | Show error, keep flag |

## Effects plan (React Query)

### Save and navigate mutation
```typescript
const useSaveAndNavigateMutation = (dispatch: Dispatch) => {
  return useMutation({
    mutationFn: ({ formId, data, navigateTo }: { 
      formId: string
      data: FormData
      navigateTo: string 
    }) => saveForm(formId, data),
    
    onMutate: async ({ formId, data }) => {
      // Cancel in-flight queries
      await queryClient.cancelQueries(['form', formId])
      
      // Optimistic update
      const previousData = queryClient.getQueryData(['form', formId])
      queryClient.setQueryData(['form', formId], data)
      
      // Transition to saving
      dispatch({ type: 'STARTED_SAVE_AND_NAVIGATE' })
      
      return { previousData }
    },
    
    onSuccess: (_, variables) => {
      dispatch({ 
        type: 'SAVED_AND_NAVIGATING', 
        payload: { navigateTo: variables.navigateTo } 
      })
      
      // Actually navigate after successful save
      router.push(variables.navigateTo)
    },
    
    onError: (error, variables, context) => {
      // Rollback optimistic update
      if (context?.previousData) {
        queryClient.setQueryData(['form', variables.formId], context.previousData)
      }
      
      dispatch({ 
        type: 'SAVE_AND_NAVIGATE_FAILED', 
        payload: { error: error.message } 
      })
    },
    
    onSettled: (result, error, variables) => {
      queryClient.invalidateQueries(['form', variables.formId])
    }
  })
}
```

## Selectors

```typescript
const selectors = {
  // Should navigation be blocked?
  shouldBlockNavigation: (state: WorkflowState) => {
    return state.hasUnsavedChanges && 
           state.navigation.phase === NavigationPhase.BLOCKED
  },
  
  // Current navigation decision context
  navigationContext: (state: WorkflowState) => {
    const { phase, pendingNavigation } = state.navigation
    
    if (phase !== NavigationPhase.CONFIRMING || !pendingNavigation) {
      return null
    }
    
    return {
      destination: pendingNavigation.to,
      trigger: pendingNavigation.trigger,
      hasUnsavedChanges: state.hasUnsavedChanges,
      canSave: state.phase !== WorkflowPhase.SAVING
    }
  },
  
  // Dialog configuration
  dialogConfig: (state: WorkflowState) => {
    const context = selectors.navigationContext(state)
    
    if (!context) return null
    
    const baseOptions = [
      { label: 'Stay', action: 'stay', variant: 'secondary' as const }
    ]
    
    if (context.canSave) {
      baseOptions.push({ 
        label: 'Save and Leave', 
        action: 'save_and_leave', 
        variant: 'primary' as const 
      })
    }
    
    baseOptions.push({ 
      label: 'Leave without Saving', 
      action: 'leave', 
      variant: 'secondary' as const 
    })
    
    return {
      isOpen: true,
      title: 'Unsaved Changes',
      message: `You have unsaved changes. What would you like to do before navigating to ${context.destination}?`,
      options: baseOptions
    }
  }
}
```

## Component boundaries

### NavigationBlocker
- **Owns**: Navigation blocking logic, dialog state
- **Receives**: Navigation context, dialog config
- **Renders**: Confirmation dialog
- **Cannot**: Access form data directly

### FormComponent
- **Owns**: Form field state, validation
- **Receives**: Form data, save actions
- **Renders**: Form fields, step navigation
- **Cannot**: Control navigation directly

### NavigationController (useNavigationWorkflow)
- **Owns**: Complete navigation and workflow state
- **Receives**: Router events, form data changes
- **Exposes**: Navigation actions, selectors
- **Cannot**: Render UI directly

## Implementation notes

### Router integration
```typescript
// Block navigation using React Router's useBlocker
const NavigationBlocker = () => {
  const { shouldBlockNavigation, pendingNavigation } = useNavigationWorkflow()
  const blocker = useBlocker(
    ({ currentLocation, nextLocation }) => {
      return shouldBlockNavigation && 
             currentLocation.pathname !== nextLocation.pathname
    }
  )
  
  useEffect(() => {
    if (blocker.state === 'blocked') {
      dispatch({
        type: 'NAVIGATION_BLOCKED',
        payload: {
          to: blocker.location.pathname,
          trigger: 'browser'
        }
      })
    }
  }, [blocker.state, blocker.location])
  
  const handleDecision = (decision: string) => {
    switch (decision) {
      case 'stay':
        dispatch({ type: 'USER_DECIDED_TO_STAY' })
        blocker.reset()
        break
      case 'leave':
        dispatch({ type: 'USER_DECIDED_TO_LEAVE' })
        blocker.proceed()
        break
      case 'save_and_leave':
        dispatch({ 
          type: 'USER_DECIDED_TO_SAVE_AND_LEAVE',
          payload: { navigateTo: blocker.location.pathname }
        })
        break
    }
  }
  
  const dialogConfig = useDialogConfig()
  
  return (
    <ConfirmationDialog
      config={dialogConfig}
      onDecision={handleDecision}
    />
  )
}
```

### Link click handling
```typescript
// Intercept link clicks within the app
const LinkInterceptor = ({ children }) => {
  const dispatch = useNavigationDispatch()
  
  const handleClick = (event: MouseEvent) => {
    const target = event.target as HTMLElement
    const link = target.closest('a[href]')
    
    if (link && link.getAttribute('data-external') !== 'true') {
      event.preventDefault()
      const href = link.getAttribute('href')
      
      dispatch({
        type: 'ATTEMPTED_NAVIGATION',
        payload: {
          to: href,
          trigger: 'link'
        }
      })
    }
  }
  
  useEffect(() => {
    document.addEventListener('click', handleClick)
    return () => document.removeEventListener('click', handleClick)
  }, [])
  
  return children
}
```

### Programmatic navigation
```typescript
// Wrapper for programmatic navigation
const useSafeNavigate = () => {
  const dispatch = useNavigationDispatch()
  const navigate = useNavigate()
  
  return useCallback((to: string) => {
    dispatch({
      type: 'ATTEMPTED_NAVIGATION',
      payload: {
        to,
        trigger: 'programmatic'
      }
    })
  }, [dispatch])
}
```

## Edge cases

1. **Browser refresh/back button**: Handle via beforeunload event
2. **Multiple rapid navigation attempts**: Queue or cancel previous attempts
3. **Save failure during navigation**: Show error, keep navigation blocked
4. **External links**: Allow without blocking (mark with data-external)
5. **Tab closing**: Use beforeunload event for unsaved changes warning
6. **Mobile app back button**: Integrate with platform-specific navigation APIs

## Testing considerations

1. **Navigation blocking**: Test that unsaved changes trigger blocking
2. **Dialog decisions**: Test each decision path (stay, leave, save_and_leave)
3. **Save failures**: Test error handling during save and navigate
4. **Multiple triggers**: Test browser back, link clicks, programmatic navigation
5. **Edge cases**: Test rapid navigation attempts, external links, tab closing
