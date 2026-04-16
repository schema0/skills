# AI Integration Patterns

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
