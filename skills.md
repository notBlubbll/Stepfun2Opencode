# Skills: Modifying Provider Lists in opencode.json

Guide for adding, removing, and configuring LLM providers in OpenCode's `opencode.json` config.

## File Location

The config file lives at the project root or `~/.config/opencode/opencode.json`.

## Basic Structure

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "<provider_id>": {
      "models": {
        "<model_id>": {}
      }
    }
  }
}
```

Format for selecting a model: `provider_id/model_id`

---

## Add a Custom Provider (OpenAI-compatible)

For any OpenAI-compatible API (local or remote):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "stepfun": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "StepFun2Opencode",
      "options": {
        "baseURL": "http://127.0.0.1:8080/v1"
      },
      "models": {
        "step-3.5-flash": { "name": "step-3.5-flash" },
        "step-3.7-flash": { "name": "step-3.7-flash" }
      }
    }
  }
}
```

Fields:
- `npm` — SDK package to use (`@ai-sdk/openai-compatible` for OpenAI-compatible APIs)
- `name` — Display name in the UI
- `options.baseURL` — API endpoint
- `models` — Map of model IDs to display names

---

## Set the Default Model

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "stepfun/step-3.5-flash"
}
```

---

## Full Example

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "stepfun/step-3.5-flash",
  "provider": {
    "stepfun": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "StepFun2Opencode",
      "options": {
        "baseURL": "http://127.0.0.1:8080/v1"
      },
      "models": {
        "step-3.5-flash": { "name": "step-3.5-flash" },
        "step-3.5-flash-2603": { "name": "step-3.5-flash-2603" },
        "step-3.7-flash": { "name": "step-3.7-flash" },
        "stepaudio-2.5-tts": { "name": "stepaudio-2.5-tts" },
        "stepaudio-2.5-asr": { "name": "stepaudio-2.5-asr" },
        "step-image-edit-2": { "name": "step-image-edit-2" }
      }
    }
  }
}
```

---

## Quick Reference

| Task | Key |
|------|-----|
| Add provider | `provider.<id>` |
| Add model | `provider.<id>.models.<model_id>` |
| Set base URL | `provider.<id>.options.baseURL` |
| Set display name | `provider.<id>.name` |
| Set SDK package | `provider.<id>.npm` |
| Configure model options | `provider.<id>.models.<model_id>.options` |
| Set default model | `model` (top-level) |

Source: https://opencode.ai/docs/models/#configure-models
