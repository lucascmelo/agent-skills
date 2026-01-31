# Async Validation Flow Example

## Problem statement
User input requires async validation (username availability, email uniqueness). Debouncing should not change logical state - only transitions should affect state. Validation results must be clearly communicated without blocking UI.

## State model

### Server cache (React Query)
```typescript
interface ValidationRule {
  field: string
  value: string
  isValid: boolean
  message?: string
  validatedAt: Date
}

// Owner: React Query cache (short TTL)
const useValidationQuery = (field: string, value: string) => 
  useQuery(
    ['validation', field, value],
    () => validateField(field, value),
    {
      enabled: value.length > 0,
      staleTime: 30000, // 30 seconds
      cacheTime: 60000, // 1 minute
    }
  )
```

### Client workflow state
```typescript
enum ValidationPhase {
  IDLE = 'idle',
  PENDING = 'pending',
  VALID = 'valid',
  INVALID = 'invalid',
  ERROR = 'error'
}

interface FieldValidation {
  [fieldName: string]: {
    phase: ValidationPhase
    lastValidated?: Date
    message?: string
    debouncedValue?: string
  }
}

interface WorkflowState {
  fields: FieldValidation
  canSubmit: boolean
  isSubmitting: boolean
}

// Owner: Feature controller (useValidationWorkflow hook)
```

### Ephemeral UI state
```typescript
interface FieldUIState {
  [fieldName: string]: {
    isFocused: boolean
    isBlurred: boolean
    showValidation: boolean
  }
}

// Owner: Individual form field components
```

## Transition table

| Current State | Trigger | Next State | Action |
|---------------|---------|------------|--------|
| IDLE | fieldChanged(field, value) | PENDING | Start debounced validation |
| PENDING | validationStarted(field) | PENDING | Show loading indicator |
| PENDING | validationSuccess(field) | VALID | Update field state |
| PENDING | validationInvalid(field, message) | INVALID | Update field with error |
| PENDING | validationError(field, error) | ERROR | Show error state |
| VALID | fieldChanged(field, value) | PENDING | Restart validation |
| INVALID | fieldChanged(field, value) | PENDING | Restart validation |
| ERROR | fieldChanged(field, value) | PENDING | Restart validation |

## Effects plan (React Query)

### Debounced validation trigger
```typescript
const useDebouncedValidation = (dispatch: Dispatch) => {
  const debouncedValidation = useMemo(
    () => debounce((field: string, value: string) => {
      dispatch({
        type: 'VALIDATION_STARTED',
        payload: { field, value }
      })
    }, 300),
    [dispatch]
  )
  
  return debouncedValidation
}

// In the validation workflow hook
const useValidationWorkflow = () => {
  const dispatch = useValidationDispatch()
  const debouncedValidate = useDebouncedValidation(dispatch)
  
  const validateField = useCallback((field: string, value: string) => {
    // Update debounced value immediately for UI feedback
    dispatch({
      type: 'FIELD_CHANGED',
      payload: { field, value, debouncedValue: value }
    })
    
    // Trigger debounced validation
    debouncedValidate(field, value)
  }, [dispatch, debouncedValidate])
  
  return { validateField }
}
```

### Validation query with state transitions
```typescript
const useFieldValidation = (field: string, value: string, dispatch: Dispatch) => {
  const query = useQuery(
    ['validation', field, value],
    () => validateField(field, value),
    {
      enabled: false, // Manually enabled
      
      onSuccess: (result) => {
        if (result.isValid) {
          dispatch({
            type: 'VALIDATION_SUCCESS',
            payload: { 
              field, 
              message: result.message 
            }
          })
        } else {
          dispatch({
            type: 'VALIDATION_INVALID',
            payload: { 
              field, 
              message: result.message || 'Validation failed' 
            }
          })
        }
      },
      
      onError: (error) => {
        dispatch({
          type: 'VALIDATION_ERROR',
          payload: { 
            field, 
            message: error.message || 'Validation service unavailable' 
          }
        })
      }
    }
  )
  
  // Manually trigger query when validation starts
  useEffect(() => {
    if (shouldValidate) {
      query.refetch()
    }
  }, [shouldValidate, query.refetch])
}
```

## Selectors

```typescript
const selectors = {
  // Current validation state for a field
  fieldValidation: (state: WorkflowState, fieldName: string) => {
    return state.fields[fieldName] || { phase: ValidationPhase.IDLE }
  },
  
  // Should show validation for a field?
  shouldShowValidation: (state: WorkflowState, fieldName: string, uiState: FieldUIState) => {
    const fieldState = state.fields[fieldName]
    const fieldUI = uiState[fieldName]
    
    if (!fieldState || !fieldUI) return false
    
    // Show if field is blurred and not idle
    return fieldUI.isBlurred && fieldState.phase !== ValidationPhase.IDLE
  },
  
  // Overall form validity
  isFormValid: (state: WorkflowState) => {
    return Object.values(state.fields).every(
      field => field.phase === ValidationPhase.VALID || field.phase === ValidationPhase.IDLE
    )
  },
  
  // Can submit form?
  canSubmit: (state: WorkflowState) => {
    return selectors.isFormValid(state) && !state.isSubmitting
  },
  
  // Validation message for field
  validationMessage: (state: WorkflowState, fieldName: string) => {
    const field = state.fields[fieldName]
    return field?.message
  }
}
```

## Component boundaries

### ValidatedField
- **Owns**: Focus/blur state, validation display logic
- **Receives**: Field value, validation state, validation actions
- **Renders**: Input field, validation message, loading indicator
- **Cannot**: Trigger validation directly (receives from parent)

### FormComponent
- **Owns**: Form submission state, overall validation
- **Receives**: All field validation states
- **Renders**: Form fields, submit button
- **Cannot**: Access individual field UI state

### ValidationController (useValidationWorkflow)
- **Owns**: Complete validation state, debouncing logic
- **Receives**: Field change events
- **Exposes**: Validation actions, selectors
- **Cannot**: Render UI directly

## Implementation notes

### Debouncing without state changes
```typescript
// Important: Debouncing changes UI but not logical state
const validateField = useCallback((field: string, value: string) => {
  // Update UI state immediately (ephemeral)
  setFieldUI(prev => ({
    ...prev,
    [field]: { ...prev[field], showValidation: false }
  }))
  
  // Update workflow state with debounced value
  dispatch({
    type: 'FIELD_CHANGED',
    payload: { field, value, debouncedValue: value }
  })
  
  // Debounced validation trigger
  debouncedValidate(field, value)
}, [dispatch, debouncedValidate])
```

### Validation result caching
```typescript
// Use React Query's built-in caching for validation results
const useCachedValidation = (field: string, value: string) => {
  return useQuery(
    ['validation', field, value],
    () => validateField(field, value),
    {
      staleTime: 30000, // Consider fresh for 30 seconds
      cacheTime: 60000, // Keep in cache for 1 minute
      refetchOnWindowFocus: false,
      retry: 1, // Only retry once for validation
    }
  )
}
```

### Real-time validation feedback
```typescript
// Show real-time feedback without blocking input
const ValidationFeedback = ({ fieldName, validationState, showValidation }) => {
  if (!showValidation) return null
  
  switch (validationState.phase) {
    case ValidationPhase.PENDING:
      return <LoadingIndicator size="small" />
    
    case ValidationPhase.VALID:
      return <SuccessIcon />
    
    case ValidationPhase.INVALID:
    case ValidationPhase.ERROR:
      return (
        <ErrorMessage>
          {validationState.message}
        </ErrorMessage>
      )
    
    default:
      return null
  }
}
```

### Form submission with validation
```typescript
const useFormSubmission = (dispatch: Dispatch) => {
  return useMutation({
    mutationFn: (formData: FormData) => submitForm(formData),
    
    onMutate: () => {
      dispatch({ type: 'SUBMISSION_STARTED' })
    },
    
    onSuccess: () => {
      dispatch({ type: 'SUBMISSION_SUCCESS' })
    },
    
    onError: (error) => {
      dispatch({ 
        type: 'SUBMISSION_ERROR',
        payload: { error: error.message }
      })
    },
    
    onSettled: () => {
      dispatch({ type: 'SUBMISSION_FINISHED' })
    }
  })
}
```

## Edge cases

1. **Rapid input changes**: Cancel previous validation requests
2. **Network failures**: Show error but allow retry
3. **Server validation errors**: Display field-specific errors
4. **Concurrent field validation**: Allow multiple fields to validate simultaneously
5. **Validation service downtime**: Graceful degradation with client-side validation
6. **Large forms**: Validate only visible/changed fields

## Performance considerations

1. **Debounce timing**: 200-500ms for responsive but not spammy validation
2. **Query caching**: Appropriate TTL to balance freshness and performance
3. **Request cancellation**: Abort previous validation requests
4. **Selective validation**: Only validate fields that have validation rules
5. **Batch validation**: For forms with interdependent fields

## Testing considerations

1. **Debouncing**: Test that rapid changes don't trigger excessive validations
2. **State transitions**: Test all phase transitions and edge cases
3. **Error handling**: Test network failures and server errors
4. **Caching**: Test that cached results are used appropriately
5. **UI feedback**: Test that validation feedback appears/disappears correctly
6. **Form submission**: Test that invalid forms cannot be submitted
