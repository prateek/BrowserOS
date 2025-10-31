# GPT-5 and GPT-4.1 Support - Quick Reference

## For Backend Developers

### The Problem
OpenAI's GPT-5 and GPT-4.1 models reject `max_tokens` and require `max_completion_tokens` instead.

### The Fix
Use the helper function to check which parameter to use:

```typescript
import { usesMaxCompletionTokens } from '@/settings/nxtscape_page/models_data';

// When building your API request:
const params: any = {
  model: modelId,
  messages: messages,
  temperature: 0.7,
};

// Add the correct token limit parameter
if (usesMaxCompletionTokens(providerType, modelId)) {
  params.max_completion_tokens = maxTokens;  // For GPT-5, GPT-4.1
} else {
  params.max_tokens = maxTokens;  // For older models
}

// Make the API call
const response = await fetch(apiUrl, {
  method: 'POST',
  body: JSON.stringify(params),
});
```

### Affected Models
- **GPT-5**: gpt-5, gpt-5-mini, gpt-5-nano
- **GPT-4.1**: gpt-4.1, gpt-4.1-mini, gpt-4.1-nano

### Custom Models
The function automatically detects custom model names like:
- `gpt5-custom`
- `gpt-5-fine-tuned`
- `gpt4.1-custom`
- `gpt-4.1-specialized`

### Testing
```bash
# Test with GPT-5
curl -X POST https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-5",
    "messages": [{"role": "user", "content": "test"}],
    "max_completion_tokens": 100
  }'

# Should work! ✅

# Test with GPT-4 (older model)
curl -X POST https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "test"}],
    "max_tokens": 100
  }'

# Still works! ✅
```

### Documentation
- **Full Guide**: `docs/GPT5_API_CHANGES.md`
- **Implementation**: `IMPLEMENTATION_SUMMARY.md`

### Questions?
Check the documentation or reach out on Discord/Slack.
