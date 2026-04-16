# AI Integration

AI SDK integration with ORPC for chat, prompt-response, and tool-calling features.

## Quick Start: Generate AI Features

```bash
# Full-stack chat with streaming
schema0 sandbox exec "bun run .claude/skills/ai-integration/scripts/generate.ts chat <name>"

# Simple prompt-response (no streaming)
schema0 sandbox exec "bun run .claude/skills/ai-integration/scripts/generate.ts simple <name>"

# Backend router only
schema0 sandbox exec "bun run .claude/skills/ai-integration/scripts/generate.ts router <name>"

# Tool definition for function calling
schema0 sandbox exec "bun run .claude/skills/ai-integration/scripts/generate.ts tool <name>"
```

## Prerequisites

1. **Add API key** using `manage-secrets` skill to securely add it and update `packages/auth/env.ts`
2. **Install dependencies:**
   ```bash
   schema0 sandbox exec "bun add ai @ai-sdk/openai @ai-sdk/anthropic @ai-sdk/google @orpc/ai-sdk @orpc/client"
   ```
3. **Add key to env schema** in `packages/auth/env.ts`:
   ```typescript
   // OpenAI
   OPENAI_API_KEY: z.string().optional(),
   // Anthropic
   ANTHROPIC_API_KEY: z.string().optional(),
   // Google Gemini
   GOOGLE_GENERATIVE_AI_API_KEY: z.string().optional(),
   ```

## Generated Files

| Template            | Output Location                        | Purpose                               |
| ------------------- | -------------------------------------- | ------------------------------------- |
| `ai-router.hbs`     | `packages/api/src/routers/[name].ts`   | ORPC router with AI SDK streaming     |
| `ai-chat-route.hbs` | `apps/web/src/routes/_auth.[name].tsx` | Full chat UI with streaming           |
| `ai-simple.hbs`     | `apps/web/src/routes/_auth.[name].tsx` | Simple prompt-response UI             |
| `ai-tool.hbs`       | `packages/api/src/tools/[name].ts`     | Tool definitions for function calling |

## Post-Generation Steps

1. **Register router** in `packages/api/src/routers/index.ts`:
   ```typescript
   import { assistantRouter } from "./assistant";
   export const appRouter = { assistant: assistantRouter, ... };
   ```
2. **Add route to sidebar** in `apps/web/src/components/app-sidebar.tsx`
3. **Set the API key environment variable** during build/deploy

## Chat Mode (Streaming with History)

Uses `useChat` from `@ai-sdk/react` with ORPC streaming transport:

```typescript
import { useChat } from "@ai-sdk/react";
import { eventIteratorToUnproxiedDataStream } from "@orpc/client";

export function AssistantChat() {
  const { messages, sendMessage, status } = useChat({
    transport: {
      async sendMessages(options) {
        return eventIteratorToUnproxiedDataStream(
          await orpc.assistant.chat(
            {
              messages: options.messages,
            },
            { signal: options.abortSignal },
          ),
        );
      },
      reconnectToStream(options) {
        throw new Error("Unsupported");
      },
    },
  });
  // ... UI implementation
}
```

## Simple Mode (One-Shot Response)

No streaming or message history:

```typescript
const response = await orpc.summarize.prompt({
  prompt: "Summarize this text...",
});
```

## Tool Calling

Use `implementTool` from `@orpc/ai-sdk` to create AI SDK tools from ORPC contracts:

```typescript
import { implementTool } from "@orpc/ai-sdk";
import { oc } from "@orpc/contract";

const getWeatherContract = oc
  .meta({
    [AI_SDK_TOOL_META_SYMBOL]: {
      title: "Get Weather",
    },
  })
  .route({
    summary: "Get the weather in a location",
  })
  .input(
    z.object({
      location: z.string().describe("The location to get the weather for"),
    }),
  )
  .output(
    z.object({
      location: z.string(),
      temperature: z.number(),
    }),
  );

const getWeatherTool = implementTool(getWeatherContract, {
  execute: async ({ location }) => ({
    location,
    temperature: 72,
  }),
});
```

## AI Provider Examples

### OpenAI (Default)

```typescript
import { openai } from "@ai-sdk/openai";
import { env } from "@template/auth";

const llmClient = openai({ apiKey: env.OPENAI_API_KEY });

const streamResult = streamText({
  model: llmClient("gpt-4o-mini"),
  system: "You are a helpful assistant.",
  messages: await convertToModelMessages(input.messages),
});
```

### Anthropic

```typescript
import { anthropic } from "@ai-sdk/anthropic";
import { env } from "@template/auth";

const llmClient = anthropic({ apiKey: env.ANTHROPIC_API_KEY });

const streamResult = streamText({
  model: llmClient("claude-3-5-sonnet-20241022"),
  system: "You are a helpful assistant.",
  messages: await convertToModelMessages(input.messages),
});
```

### Google Gemini

```typescript
import { google } from "@ai-sdk/google";
import { env } from "@template/auth";

const llmClient = google({ apiKey: env.GOOGLE_GENERATIVE_AI_API_KEY });

const streamResult = streamText({
  model: llmClient("gemini-1.5-flash"),
  system: "You are a helpful assistant.",
  messages: await convertToModelMessages(input.messages),
});
```

## Variable Naming Conventions

| Pattern                | Example                | Usage                                                     |
| ---------------------- | ---------------------- | --------------------------------------------------------- |
| `llmClient`            | `llmClient`            | LLM provider client (openai, anthropic, google)           |
| `streamResult`         | `streamResult`         | Result from `streamText()` for streaming                  |
| `textGenerationResult` | `textGenerationResult` | Result from `generateText()` for one-shot                 |
| `userMessage`          | `userMessage`          | User input message (avoids confusion with messages array) |
| `aiResponse`           | `aiResponse`           | AI response from simple prompt endpoint                   |

### Provider Pattern

```typescript
// Provider configuration
const llmClient = openai({ apiKey: env.OPENAI_API_KEY });

// Streaming
const streamResult = streamText({
  model: llmClient("gpt-4o-mini"),
  system: "You are a helpful assistant.",
  messages: convertToCoreMessages(input.messages),
});

// One-shot
const textGenerationResult = await generateText({
  model: llmClient("gpt-4o-mini"),
  system: input.system || "You are a helpful assistant.",
  prompt: input.prompt,
});

return {
  text: textGenerationResult.text,
  usage: textGenerationResult.usage,
  finishReason: textGenerationResult.finishReason,
};
```

## Type Safety

- NEVER use `any` type in generated code
- NEVER suppress typecheck errors with `// @ts-ignore`, `// @ts-expect-error`, `// @ts-nocheck`, or `// eslint-disable`
