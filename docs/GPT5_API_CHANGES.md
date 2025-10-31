# GPT-5 and GPT-4.1 API Changes

## Problem

OpenAI has changed the API for GPT-4.1 and GPT-5 family models. These newer models **do not accept** the `max_tokens` parameter and instead require `max_completion_tokens`.

### Affected Models

- **GPT-5 family**: `gpt-5`, `gpt-5-mini`, `gpt-5-nano`
- **GPT-4.1 family**: `gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`

### API Parameter Changes

| Model Family | Old Parameter | New Parameter |
|---|---|---|
| GPT-3.5, GPT-4 (original) | `max_tokens` | `max_tokens` |
| GPT-4.1, GPT-5 | ❌ `max_tokens` | ✅ `max_completion_tokens` |

## Solution

### 1. Model Metadata

We've updated `models_data.ts` to include a flag indicating which models use the new parameter:

```typescript
export interface ModelInfo {
  model_id: string;
  context_length: number;
  uses_max_completion_tokens?: boolean;  // New field
}
```

Example model entries:
```typescript
{ model_id: 'gpt-5', context_length: 400000, uses_max_completion_tokens: true },
{ model_id: 'gpt-4.1', context_length: 1047576, uses_max_completion_tokens: true },
```

### 2. Helper Function

Use the `usesMaxCompletionTokens()` function to check if a model needs the new parameter:

```typescript
import { usesMaxCompletionTokens } from './models_data';

const providerType = 'openai';
const modelId = 'gpt-5';

if (usesMaxCompletionTokens(providerType, modelId)) {
  // Use max_completion_tokens
  apiParams.max_completion_tokens = 4096;
} else {
  // Use max_tokens (for older models)
  apiParams.max_tokens = 4096;
}
```

### 3. Implementation in API Calls

When making OpenAI API calls, use this pattern:

```typescript
// Example: Building API request payload
function buildChatCompletionRequest(
  modelId: string, 
  messages: Array<Message>,
  maxTokens: number,
  providerType: string = 'openai'
): ChatCompletionRequest {
  const request: ChatCompletionRequest = {
    model: modelId,
    messages: messages,
    // ... other parameters
  };

  // Use the correct parameter based on model
  if (usesMaxCompletionTokens(providerType, modelId)) {
    request.max_completion_tokens = maxTokens;
  } else {
    request.max_tokens = maxTokens;
  }

  return request;
}
```

### 4. Fallback Logic

The `usesMaxCompletionTokens()` function includes fallback logic for custom models:

```typescript
export function usesMaxCompletionTokens(providerType: string, modelId: string): boolean {
  // Only OpenAI and OpenAI-compatible providers may have this requirement
  if (providerType !== 'openai' && providerType !== 'openai_compatible' && providerType !== 'openrouter') {
    return false;
  }
  
  const models = getModelsForProvider(providerType);
  const model = models.find(m => m.model_id === modelId);
  
  // Check explicit flag first
  if (model?.uses_max_completion_tokens !== undefined) {
    return model.uses_max_completion_tokens;
  }
  
  // Fallback: Check if model ID matches GPT-4.1 or GPT-5 patterns
  const modelIdLower = modelId.toLowerCase();
  return modelIdLower.includes('gpt-5') || 
         modelIdLower.includes('gpt-4.1') ||
         modelIdLower.startsWith('gpt5') ||
         modelIdLower.startsWith('gpt4.1');
}
```

## Testing

### Manual Testing

1. Configure BrowserOS to use a GPT-5 or GPT-4.1 model
2. Make an API call with the assistant
3. Verify that the request uses `max_completion_tokens` instead of `max_tokens`
4. Confirm the API call succeeds without errors

### Unit Tests

If you're implementing this in the agent code, add tests like:

```typescript
import { usesMaxCompletionTokens } from '@/lib/models_data';

describe('usesMaxCompletionTokens', () => {
  test('returns true for GPT-5 models', () => {
    expect(usesMaxCompletionTokens('openai', 'gpt-5')).toBe(true);
    expect(usesMaxCompletionTokens('openai', 'gpt-5-mini')).toBe(true);
    expect(usesMaxCompletionTokens('openai', 'gpt-5-nano')).toBe(true);
  });

  test('returns true for GPT-4.1 models', () => {
    expect(usesMaxCompletionTokens('openai', 'gpt-4.1')).toBe(true);
    expect(usesMaxCompletionTokens('openai', 'gpt-4.1-mini')).toBe(true);
  });

  test('returns false for older OpenAI models', () => {
    expect(usesMaxCompletionTokens('openai', 'gpt-4')).toBe(false);
    expect(usesMaxCompletionTokens('openai', 'gpt-4-turbo')).toBe(false);
    expect(usesMaxCompletionTokens('openai', 'gpt-3.5-turbo')).toBe(false);
  });

  test('returns false for non-OpenAI providers', () => {
    expect(usesMaxCompletionTokens('anthropic', 'claude-3-opus')).toBe(false);
    expect(usesMaxCompletionTokens('google_gemini', 'gemini-pro')).toBe(false);
  });

  test('handles custom model names with pattern matching', () => {
    expect(usesMaxCompletionTokens('openai', 'gpt5-custom')).toBe(true);
    expect(usesMaxCompletionTokens('openai', 'gpt4.1-custom')).toBe(true);
  });
});
```

## Migration Guide

If you have existing code that sets `max_tokens` for all OpenAI models, update it as follows:

### Before
```typescript
const response = await openai.chat.completions.create({
  model: selectedModel,
  messages: messages,
  max_tokens: 4096,  // This will fail for GPT-5 and GPT-4.1
});
```

### After
```typescript
import { usesMaxCompletionTokens } from './models_data';

const requestParams: any = {
  model: selectedModel,
  messages: messages,
};

// Use the correct parameter based on model
if (usesMaxCompletionTokens('openai', selectedModel)) {
  requestParams.max_completion_tokens = 4096;
} else {
  requestParams.max_tokens = 4096;
}

const response = await openai.chat.completions.create(requestParams);
```

## References

- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference/chat/create)
- [OpenAI Migration Guide](https://platform.openai.com/docs/guides/migration)
- BrowserOS Models Data: `packages/browseros/chromium_patches/chrome/browser/resources/settings/nxtscape_page/models_data.ts`

## Troubleshooting

### Error: Invalid parameter 'max_tokens'

If you see this error when using GPT-5 or GPT-4.1 models:
- Check that you're using `max_completion_tokens` instead of `max_tokens`
- Verify the model ID is being detected correctly
- Check that the `usesMaxCompletionTokens()` function is being called

### Models not detected correctly

If a custom model isn't being detected:
1. Add it to the `MODELS_DATA` in `models_data.ts` with the `uses_max_completion_tokens` flag
2. Or ensure the model ID contains 'gpt-5' or 'gpt-4.1' for automatic detection

## Future Considerations

- OpenAI may add more models that require `max_completion_tokens`
- Monitor OpenAI's API changelog for additional parameter changes
- Consider adding a configuration option to override the detection logic for edge cases
