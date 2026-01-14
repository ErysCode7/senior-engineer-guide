# OpenAI and Anthropic API Integration

## Overview

Integrating AI models like OpenAI's GPT-4 and Anthropic's Claude into your applications opens up powerful capabilities for natural language understanding, generation, and analysis. This tutorial covers best practices for API integration, prompt engineering, streaming responses, and cost optimization.

## Use Cases

- **Chatbots and Virtual Assistants**: Conversational AI for customer support
- **Content Generation**: Blog posts, product descriptions, marketing copy
- **Code Assistance**: Code generation, debugging, and explanations
- **Data Analysis**: Summarization, extraction, and insights from text
- **Translation and Localization**: Multi-language content processing

## Setting Up OpenAI Integration

### Installation and Configuration

```bash
npm install openai
```

```typescript
// src/config/openai.config.ts
import { OpenAI } from "openai";

export const openaiConfig = {
  apiKey: process.env.OPENAI_API_KEY,
  organization: process.env.OPENAI_ORG_ID, // Optional
};

export const openai = new OpenAI({
  apiKey: openaiConfig.apiKey,
  organization: openaiConfig.organization,
});
```

### Basic Chat Completion

```typescript
// src/services/openai.service.ts
import { openai } from "../config/openai.config";

export class OpenAIService {
  async generateChatCompletion(
    messages: Array<{ role: "system" | "user" | "assistant"; content: string }>,
    options?: {
      model?: string;
      temperature?: number;
      maxTokens?: number;
    }
  ) {
    try {
      const completion = await openai.chat.completions.create({
        model: options?.model || "gpt-4-turbo-preview",
        messages,
        temperature: options?.temperature || 0.7,
        max_tokens: options?.maxTokens || 1000,
      });

      return {
        content: completion.choices[0].message.content,
        usage: completion.usage,
        model: completion.model,
      };
    } catch (error) {
      console.error("OpenAI API Error:", error);
      throw new Error("Failed to generate completion");
    }
  }

  async generateSimpleResponse(prompt: string): Promise<string> {
    const response = await this.generateChatCompletion([
      {
        role: "system",
        content:
          "You are a helpful assistant that provides clear, concise answers.",
      },
      {
        role: "user",
        content: prompt,
      },
    ]);

    return response.content;
  }
}

// Usage example
const response = await openaiService.generateSimpleResponse(
  "Explain quantum computing in simple terms"
);
```

### Streaming Responses

```typescript
// src/services/openai-stream.service.ts
import { openai } from "../config/openai.config";

export class OpenAIStreamService {
  async *streamChatCompletion(
    messages: Array<{ role: "system" | "user" | "assistant"; content: string }>
  ): AsyncGenerator<string> {
    const stream = await openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages,
      stream: true,
    });

    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content || "";
      if (content) {
        yield content;
      }
    }
  }
}

// Controller implementation for streaming
import { Response } from "express";

export class AIController {
  constructor(private readonly streamService: OpenAIStreamService) {}

  @Post("chat/stream")
  async streamChat(@Body() body: { messages: any[] }, @Res() res: Response) {
    res.setHeader("Content-Type", "text/event-stream");
    res.setHeader("Cache-Control", "no-cache");
    res.setHeader("Connection", "keep-alive");

    try {
      for await (const chunk of this.streamService.streamChatCompletion(
        body.messages
      )) {
        res.write(`data: ${JSON.stringify({ content: chunk })}\n\n`);
      }

      res.write("data: [DONE]\n\n");
      res.end();
    } catch (error) {
      res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
      res.end();
    }
  }
}
```

### Function Calling (Tools)

```typescript
// src/services/openai-functions.service.ts
import { openai } from "../config/openai.config";

interface FunctionDefinition {
  name: string;
  description: string;
  parameters: {
    type: "object";
    properties: Record<string, any>;
    required: string[];
  };
}

export class OpenAIFunctionsService {
  private functions: FunctionDefinition[] = [
    {
      name: "get_weather",
      description: "Get the current weather for a location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g. San Francisco, CA",
          },
          unit: {
            type: "string",
            enum: ["celsius", "fahrenheit"],
            description: "The temperature unit",
          },
        },
        required: ["location"],
      },
    },
    {
      name: "search_database",
      description: "Search the product database",
      parameters: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "The search query",
          },
          category: {
            type: "string",
            description: "Product category to filter by",
          },
        },
        required: ["query"],
      },
    },
  ];

  async chatWithFunctions(userMessage: string) {
    const messages = [
      {
        role: "system" as const,
        content: "You are a helpful assistant that can access external tools.",
      },
      {
        role: "user" as const,
        content: userMessage,
      },
    ];

    // First API call
    let response = await openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages,
      tools: this.functions.map((func) => ({
        type: "function" as const,
        function: func,
      })),
      tool_choice: "auto",
    });

    let assistantMessage = response.choices[0].message;

    // Check if the model wants to call a function
    if (assistantMessage.tool_calls) {
      messages.push(assistantMessage);

      // Execute each function call
      for (const toolCall of assistantMessage.tool_calls) {
        const functionName = toolCall.function.name;
        const functionArgs = JSON.parse(toolCall.function.arguments);

        // Execute the actual function
        const functionResult = await this.executeFunction(
          functionName,
          functionArgs
        );

        // Add function result to messages
        messages.push({
          role: "tool" as const,
          content: JSON.stringify(functionResult),
          tool_call_id: toolCall.id,
        });
      }

      // Get final response with function results
      response = await openai.chat.completions.create({
        model: "gpt-4-turbo-preview",
        messages,
      });

      assistantMessage = response.choices[0].message;
    }

    return {
      content: assistantMessage.content,
      functionsCalled:
        assistantMessage.tool_calls?.map((tc) => tc.function.name) || [],
    };
  }

  private async executeFunction(name: string, args: any): Promise<any> {
    switch (name) {
      case "get_weather":
        return this.getWeather(args.location, args.unit);
      case "search_database":
        return this.searchDatabase(args.query, args.category);
      default:
        return { error: "Unknown function" };
    }
  }

  private async getWeather(location: string, unit: string = "celsius") {
    // In production, call a real weather API
    return {
      location,
      temperature: 22,
      unit,
      conditions: "Sunny",
    };
  }

  private async searchDatabase(query: string, category?: string) {
    // In production, query your actual database
    return {
      results: [
        { id: 1, name: "Product A", category: "Electronics" },
        { id: 2, name: "Product B", category: "Electronics" },
      ],
    };
  }
}

// Usage
const result = await openAIFunctionsService.chatWithFunctions(
  "What's the weather like in San Francisco?"
);
```

## Anthropic Claude Integration

### Setup

```bash
npm install @anthropic-ai/sdk
```

```typescript
// src/config/anthropic.config.ts
import Anthropic from "@anthropic-ai/sdk";

export const anthropicConfig = {
  apiKey: process.env.ANTHROPIC_API_KEY,
};

export const anthropic = new Anthropic({
  apiKey: anthropicConfig.apiKey,
});
```

### Basic Message Generation

```typescript
// src/services/anthropic.service.ts
import { anthropic } from "../config/anthropic.config";

export class AnthropicService {
  async generateMessage(
    prompt: string,
    options?: {
      model?: string;
      maxTokens?: number;
      temperature?: number;
      system?: string;
    }
  ) {
    try {
      const message = await anthropic.messages.create({
        model: options?.model || "claude-3-opus-20240229",
        max_tokens: options?.maxTokens || 1024,
        temperature: options?.temperature || 0.7,
        system: options?.system || "You are a helpful AI assistant.",
        messages: [
          {
            role: "user",
            content: prompt,
          },
        ],
      });

      return {
        content:
          message.content[0].type === "text" ? message.content[0].text : "",
        usage: message.usage,
        model: message.model,
        stopReason: message.stop_reason,
      };
    } catch (error) {
      console.error("Anthropic API Error:", error);
      throw new Error("Failed to generate message");
    }
  }

  async generateConversation(
    messages: Array<{ role: "user" | "assistant"; content: string }>,
    systemPrompt: string = "You are a helpful AI assistant."
  ) {
    const response = await anthropic.messages.create({
      model: "claude-3-opus-20240229",
      max_tokens: 2048,
      system: systemPrompt,
      messages,
    });

    return {
      content:
        response.content[0].type === "text" ? response.content[0].text : "",
      usage: response.usage,
    };
  }
}
```

### Streaming with Claude

```typescript
// src/services/anthropic-stream.service.ts
import { anthropic } from "../config/anthropic.config";

export class AnthropicStreamService {
  async *streamMessage(prompt: string): AsyncGenerator<string> {
    const stream = await anthropic.messages.create({
      model: "claude-3-opus-20240229",
      max_tokens: 1024,
      messages: [{ role: "user", content: prompt }],
      stream: true,
    });

    for await (const event of stream) {
      if (
        event.type === "content_block_delta" &&
        event.delta.type === "text_delta"
      ) {
        yield event.delta.text;
      }
    }
  }
}
```

## Prompt Engineering Best Practices

### System Prompts

```typescript
// src/prompts/system-prompts.ts
export const systemPrompts = {
  customer_support: `You are a friendly customer support agent for TechCorp.
    - Always be polite and professional
    - Provide accurate information based on our documentation
    - If you don't know something, admit it and offer to escalate
    - Keep responses concise but helpful
    - Use bullet points for multiple steps`,

  code_assistant: `You are an expert software engineer specializing in TypeScript and Node.js.
    - Provide clean, well-documented code examples
    - Explain your reasoning
    - Follow best practices and design patterns
    - Consider performance and security
    - Suggest tests when appropriate`,

  content_writer: `You are a professional content writer.
    - Write in a clear, engaging style
    - Use proper grammar and punctuation
    - Match the tone to the target audience
    - Include relevant keywords naturally
    - Structure content with headers and paragraphs`,
};
```

### Structured Prompts

```typescript
// src/services/prompt-builder.service.ts

interface PromptTemplate {
  system: string;
  userTemplate: string;
}

export class PromptBuilderService {
  private templates: Record<string, PromptTemplate> = {
    summarize: {
      system:
        "You are an expert at summarizing long texts into concise summaries.",
      userTemplate: `Please summarize the following text in {{length}} sentences:

{{text}}

Focus on the main points and key takeaways.`,
    },

    extract_info: {
      system: "You extract structured information from unstructured text.",
      userTemplate: `Extract the following information from the text below:
{{fields}}

Text:
{{text}}

Return the information in JSON format.`,
    },

    generate_code: {
      system: "You are an expert programmer who writes clean, efficient code.",
      userTemplate: `Generate {{language}} code for the following requirement:

{{requirement}}

Include:
- Type definitions
- Error handling
- Comments explaining key parts
- Example usage`,
    },
  };

  buildPrompt(templateName: string, variables: Record<string, string>): string {
    const template = this.templates[templateName];
    if (!template) {
      throw new Error(`Template ${templateName} not found`);
    }

    let prompt = template.userTemplate;

    // Replace variables
    for (const [key, value] of Object.entries(variables)) {
      prompt = prompt.replace(new RegExp(`{{${key}}}`, "g"), value);
    }

    return prompt;
  }

  getSystemPrompt(templateName: string): string {
    return this.templates[templateName]?.system || "";
  }
}

// Usage
const prompt = promptBuilder.buildPrompt("summarize", {
  length: "3",
  text: "Long article text here...",
});
```

### Few-Shot Learning

```typescript
// src/services/few-shot.service.ts
import { openai } from "../config/openai.config";

export class FewShotService {
  async classifyWithExamples(text: string, category: string) {
    const examples = {
      sentiment: [
        { input: "I love this product!", output: "positive" },
        { input: "This is terrible.", output: "negative" },
        { input: "It works okay.", output: "neutral" },
      ],
      intent: [
        { input: "What are your business hours?", output: "hours_inquiry" },
        { input: "I want to return my order", output: "return_request" },
        { input: "How do I reset my password?", output: "technical_support" },
      ],
    };

    const categoryExamples = examples[category] || [];

    const messages = [
      {
        role: "system" as const,
        content: `You are a classifier. Analyze the input and return only the classification label.`,
      },
      // Add examples
      ...categoryExamples.flatMap((ex) => [
        { role: "user" as const, content: ex.input },
        { role: "assistant" as const, content: ex.output },
      ]),
      // Add actual query
      { role: "user" as const, content: text },
    ];

    const response = await openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages,
      temperature: 0.3, // Lower temperature for consistent classification
      max_tokens: 10,
    });

    return response.choices[0].message.content?.trim();
  }
}
```

## Cost Optimization Strategies

### Token Counting and Limiting

```typescript
// src/services/token-counter.service.ts
import { encode } from "gpt-tokenizer";

export class TokenCounterService {
  countTokens(text: string): number {
    return encode(text).length;
  }

  truncateToTokenLimit(text: string, maxTokens: number): string {
    const tokens = encode(text);

    if (tokens.length <= maxTokens) {
      return text;
    }

    // Truncate and decode back to text
    const truncated = tokens.slice(0, maxTokens);
    return this.decodeTokens(truncated);
  }

  private decodeTokens(tokens: number[]): string {
    // Simple approximation - in production use proper decoder
    return tokens.map((t) => String.fromCharCode(t)).join("");
  }

  estimateCost(
    inputTokens: number,
    outputTokens: number,
    model: string
  ): number {
    const pricing = {
      "gpt-4-turbo-preview": { input: 0.01, output: 0.03 }, // per 1K tokens
      "gpt-3.5-turbo": { input: 0.0005, output: 0.0015 },
      "claude-3-opus-20240229": { input: 0.015, output: 0.075 },
      "claude-3-sonnet-20240229": { input: 0.003, output: 0.015 },
    };

    const modelPricing = pricing[model] || pricing["gpt-3.5-turbo"];

    return (
      (inputTokens / 1000) * modelPricing.input +
      (outputTokens / 1000) * modelPricing.output
    );
  }
}
```

### Caching Responses

```typescript
// src/services/ai-cache.service.ts
import { Redis } from 'ioredis';
import * as crypto from 'crypto';

export class AICacheService {
  private redis: Redis;

  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
    });
  }

  private generateCacheKey(prompt: string, model: string): string {
    const hash = crypto
      .createHash('sha256')
      .update(`${model}:${prompt}`)
      .digest('hex');
    return `ai:cache:${hash}`;
  }

  async getCached(prompt: string, model: string): Promise<string | null> {
    const key = this.generateCacheKey(prompt, model);
    return this.redis.get(key);
  }

  async setCached(
    prompt: string,
    model: string,
    response: string,
    ttlSeconds: number = 3600,
  ): Promise<void> {
    const key = this.generateCacheKey(prompt, model);
    await this.redis.setex(key, ttlSeconds, response);
  }

  async getOrGenerate(
    prompt: string,
    model: string,
    generateFn: () => Promise<string>,
  ): Promise<{ response: string; fromCache: boolean }> {
    // Check cache first
    const cached = await this.getCached(prompt, model);

    if (cached) {
      return { response: cached, fromCache: true };
    }

    // Generate if not cached
    const response = await generateFn();

    // Cache for future use
    await this.setCached(prompt, model, response);

    return { response, fromCache: false };
  }
}

// Usage in service
async generateWithCache(prompt: string) {
  return this.cacheService.getOrGenerate(
    prompt,
    'gpt-4-turbo-preview',
    async () => {
      const result = await this.openaiService.generateSimpleResponse(prompt);
      return result;
    },
  );
}
```

### Model Selection Strategy

```typescript
// src/services/model-selector.service.ts

interface ModelConfig {
  name: string;
  costPerToken: number;
  quality: number;
  speed: number;
}

export class ModelSelectorService {
  private models: Record<string, ModelConfig> = {
    "gpt-4-turbo": {
      name: "gpt-4-turbo-preview",
      costPerToken: 0.01,
      quality: 10,
      speed: 5,
    },
    "gpt-3.5-turbo": {
      name: "gpt-3.5-turbo",
      costPerToken: 0.0005,
      quality: 7,
      speed: 9,
    },
    "claude-opus": {
      name: "claude-3-opus-20240229",
      costPerToken: 0.015,
      quality: 10,
      speed: 6,
    },
    "claude-sonnet": {
      name: "claude-3-sonnet-20240229",
      costPerToken: 0.003,
      quality: 8,
      speed: 8,
    },
  };

  selectModel(
    taskComplexity: "simple" | "medium" | "complex",
    prioritize: "cost" | "quality" | "speed" = "balanced"
  ): string {
    // Simple tasks can use cheaper models
    if (taskComplexity === "simple") {
      return this.models["gpt-3.5-turbo"].name;
    }

    // Complex tasks need high quality
    if (taskComplexity === "complex") {
      return prioritize === "cost"
        ? this.models["claude-sonnet"].name
        : this.models["gpt-4-turbo"].name;
    }

    // Medium complexity - balance based on priority
    if (prioritize === "cost") {
      return this.models["gpt-3.5-turbo"].name;
    } else if (prioritize === "speed") {
      return this.models["gpt-3.5-turbo"].name;
    } else {
      return this.models["claude-sonnet"].name;
    }
  }
}
```

## Error Handling and Retry Logic

```typescript
// src/services/ai-resilience.service.ts

export class AIResilienceService {
  async withRetry<T>(
    operation: () => Promise<T>,
    maxRetries: number = 3,
    delayMs: number = 1000
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;

        // Don't retry on certain errors
        if (this.isNonRetryableError(error)) {
          throw error;
        }

        if (attempt < maxRetries) {
          // Exponential backoff
          const delay = delayMs * Math.pow(2, attempt - 1);
          await this.sleep(delay);
        }
      }
    }

    throw lastError;
  }

  private isNonRetryableError(error: any): boolean {
    // Don't retry on invalid requests, auth errors, etc.
    const nonRetryableStatusCodes = [400, 401, 403, 404];
    return nonRetryableStatusCodes.includes(error.status);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}

// Usage
const result = await resilienceService.withRetry(
  () => openaiService.generateSimpleResponse(prompt),
  3,
  1000
);
```

## Best Practices

### 1. **Always Set Token Limits**

Prevent runaway costs by setting `max_tokens`.

### 2. **Use Appropriate Models**

Don't use GPT-4 for simple tasks that GPT-3.5 can handle.

### 3. **Implement Caching**

Cache identical or similar requests to reduce API calls.

### 4. **Monitor Usage**

Track token usage and costs in real-time.

### 5. **Handle Errors Gracefully**

Implement retry logic with exponential backoff.

### 6. **Optimize Prompts**

Clear, concise prompts reduce token usage and improve results.

### 7. **Use Streaming for Long Responses**

Improve UX by streaming responses to users.

### 8. **Validate Inputs**

Sanitize user inputs before sending to AI APIs.

## Summary

Integrating OpenAI and Anthropic APIs enables powerful AI features in your applications. Focus on:

- Proper API setup and configuration
- Effective prompt engineering
- Streaming for better UX
- Cost optimization through caching and model selection
- Robust error handling
- Token management
- Security and input validation

These practices ensure reliable, cost-effective AI integration in production systems.
