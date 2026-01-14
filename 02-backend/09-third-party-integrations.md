# Third-Party API Integrations

## Overview

Third-party API integrations enable your application to leverage external services and platforms, extending functionality without building everything from scratch. Proper integration patterns ensure reliability, security, and maintainability when working with external APIs.

**Key Concepts:**

- **REST API Integration**: HTTP-based communication with external services
- **Webhooks**: Event-driven callbacks from external services
- **OAuth 2.0**: Secure authorization for accessing user data
- **API Clients**: Typed SDKs and HTTP clients for API communication
- **Retry Logic**: Handling transient failures and rate limits

## Practical Use Cases

### 1. **Payment Processing**

Financial transactions

- Stripe, PayPal, Square
- Subscription management
- Invoice generation
- Refund processing

### 2. **Communication Services**

Messaging and notifications

- SendGrid, Twilio (covered in email/SMS tutorial)
- Slack, Discord integrations
- Push notifications (Firebase, OneSignal)
- Video calling (Twilio, Agora)

### 3. **Social Media Integration**

User engagement and marketing

- Facebook, Twitter, LinkedIn APIs
- Social login (OAuth)
- Content sharing
- Analytics tracking

### 4. **Cloud Storage**

File management

- AWS S3, Google Cloud Storage
- Cloudinary, Uploadcare
- CDN integration

### 5. **Analytics & Monitoring**

Data collection and insights

- Google Analytics
- Mixpanel, Amplitude
- Sentry (error tracking)
- LogRocket (session replay)

### 6. **CRM & Marketing**

Customer relationship management

- Salesforce, HubSpot
- Mailchimp, Constant Contact
- Intercom, Zendesk

## Step-by-Step Implementation

### 1. Install Dependencies

```bash
# HTTP clients
npm install axios @nestjs/axios

# Type safety
npm install -D @types/node

# Optional: SDK packages
npm install @slack/web-api @octokit/rest stripe
```

### 2. HTTP Client Service (Base Class)

```typescript
// common/http-client.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { HttpService } from "@nestjs/axios";
import { AxiosError, AxiosRequestConfig, AxiosResponse } from "axios";
import { firstValueFrom, retry, catchError, throwError } from "rxjs";

export interface RetryConfig {
  maxRetries: number;
  retryDelay: number;
  retryableStatuses: number[];
}

@Injectable()
export class HttpClientService {
  private readonly logger = new Logger(HttpClientService.name);

  constructor(private readonly httpService: HttpService) {}

  async get<T>(
    url: string,
    config?: AxiosRequestConfig
  ): Promise<AxiosResponse<T>> {
    return this.request<T>({ method: "GET", url, ...config });
  }

  async post<T>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<AxiosResponse<T>> {
    return this.request<T>({ method: "POST", url, data, ...config });
  }

  async put<T>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<AxiosResponse<T>> {
    return this.request<T>({ method: "PUT", url, data, ...config });
  }

  async delete<T>(
    url: string,
    config?: AxiosRequestConfig
  ): Promise<AxiosResponse<T>> {
    return this.request<T>({ method: "DELETE", url, ...config });
  }

  private async request<T>(
    config: AxiosRequestConfig,
    retryConfig?: RetryConfig
  ): Promise<AxiosResponse<T>> {
    const defaultRetryConfig: RetryConfig = {
      maxRetries: 3,
      retryDelay: 1000,
      retryableStatuses: [408, 429, 500, 502, 503, 504],
    };

    const finalRetryConfig = { ...defaultRetryConfig, ...retryConfig };

    this.logger.debug(`Request: ${config.method} ${config.url}`);

    return firstValueFrom(
      this.httpService.request<T>(config).pipe(
        retry({
          count: finalRetryConfig.maxRetries,
          delay: (error: AxiosError, retryCount) => {
            const status = error.response?.status;

            if (status && finalRetryConfig.retryableStatuses.includes(status)) {
              const delay =
                finalRetryConfig.retryDelay * Math.pow(2, retryCount - 1);
              this.logger.warn(
                `Retrying request (${retryCount}/${finalRetryConfig.maxRetries}) after ${delay}ms`
              );
              return new Promise((resolve) => setTimeout(resolve, delay));
            }

            throw error;
          },
        }),
        catchError((error: AxiosError) => {
          this.logger.error(
            `Request failed: ${error.message}`,
            error.response?.data
          );
          return throwError(() => error);
        })
      )
    );
  }
}
```

### 3. GitHub API Integration

```typescript
// github/github.service.ts
import { Injectable, HttpException, HttpStatus } from "@nestjs/common";
import { HttpClientService } from "../common/http-client.service";
import { ConfigService } from "@nestjs/config";

export interface GitHubUser {
  login: string;
  id: number;
  name: string;
  email: string;
  avatar_url: string;
  bio: string;
  public_repos: number;
  followers: number;
}

export interface GitHubRepo {
  id: number;
  name: string;
  full_name: string;
  description: string;
  html_url: string;
  stargazers_count: number;
  language: string;
  updated_at: string;
}

@Injectable()
export class GitHubService {
  private readonly baseURL = "https://api.github.com";
  private readonly token: string;

  constructor(
    private readonly httpClient: HttpClientService,
    private readonly configService: ConfigService
  ) {
    this.token = this.configService.get<string>("GITHUB_TOKEN");
  }

  private getHeaders() {
    return {
      Authorization: `Bearer ${this.token}`,
      Accept: "application/vnd.github.v3+json",
    };
  }

  async getUserProfile(username: string): Promise<GitHubUser> {
    try {
      const response = await this.httpClient.get<GitHubUser>(
        `${this.baseURL}/users/${username}`,
        { headers: this.getHeaders() }
      );

      return response.data;
    } catch (error) {
      if (error.response?.status === 404) {
        throw new HttpException("User not found", HttpStatus.NOT_FOUND);
      }
      throw new HttpException(
        "Failed to fetch GitHub user",
        HttpStatus.BAD_GATEWAY
      );
    }
  }

  async getUserRepositories(username: string): Promise<GitHubRepo[]> {
    try {
      const response = await this.httpClient.get<GitHubRepo[]>(
        `${this.baseURL}/users/${username}/repos`,
        {
          headers: this.getHeaders(),
          params: {
            sort: "updated",
            per_page: 100,
          },
        }
      );

      return response.data;
    } catch (error) {
      throw new HttpException(
        "Failed to fetch repositories",
        HttpStatus.BAD_GATEWAY
      );
    }
  }

  async createRepository(
    name: string,
    description: string,
    isPrivate: boolean = false
  ): Promise<GitHubRepo> {
    try {
      const response = await this.httpClient.post<GitHubRepo>(
        `${this.baseURL}/user/repos`,
        {
          name,
          description,
          private: isPrivate,
          auto_init: true,
        },
        { headers: this.getHeaders() }
      );

      return response.data;
    } catch (error) {
      throw new HttpException(
        "Failed to create repository",
        HttpStatus.BAD_GATEWAY
      );
    }
  }

  async searchRepositories(query: string, limit: number = 10) {
    try {
      const response = await this.httpClient.get(
        `${this.baseURL}/search/repositories`,
        {
          headers: this.getHeaders(),
          params: {
            q: query,
            per_page: limit,
            sort: "stars",
          },
        }
      );

      return response.data.items;
    } catch (error) {
      throw new HttpException(
        "Failed to search repositories",
        HttpStatus.BAD_GATEWAY
      );
    }
  }
}
```

### 4. Slack Integration

```typescript
// slack/slack.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { WebClient } from "@slack/web-api";
import { ConfigService } from "@nestjs/config";

export interface SlackMessage {
  channel: string;
  text: string;
  blocks?: any[];
  thread_ts?: string;
}

@Injectable()
export class SlackService {
  private readonly logger = new Logger(SlackService.name);
  private readonly client: WebClient;

  constructor(private readonly configService: ConfigService) {
    const token = this.configService.get<string>("SLACK_BOT_TOKEN");
    this.client = new WebClient(token);
  }

  async sendMessage(message: SlackMessage) {
    try {
      const result = await this.client.chat.postMessage({
        channel: message.channel,
        text: message.text,
        blocks: message.blocks,
        thread_ts: message.thread_ts,
      });

      this.logger.log(`Message sent to ${message.channel}`);
      return result;
    } catch (error) {
      this.logger.error(`Failed to send Slack message: ${error.message}`);
      throw error;
    }
  }

  async sendFormattedMessage(channel: string, title: string, fields: any[]) {
    const blocks = [
      {
        type: "header",
        text: {
          type: "plain_text",
          text: title,
        },
      },
      {
        type: "section",
        fields: fields.map((field) => ({
          type: "mrkdwn",
          text: `*${field.name}:*\n${field.value}`,
        })),
      },
    ];

    return this.sendMessage({
      channel,
      text: title,
      blocks,
    });
  }

  async sendErrorAlert(channel: string, error: Error, context?: any) {
    const blocks = [
      {
        type: "header",
        text: {
          type: "plain_text",
          text: "🚨 Error Alert",
          emoji: true,
        },
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: `*Error:*\n${error.message}`,
          },
          {
            type: "mrkdwn",
            text: `*Time:*\n${new Date().toISOString()}`,
          },
        ],
      },
    ];

    if (context) {
      blocks.push({
        type: "section",
        text: {
          type: "mrkdwn",
          text: `*Context:*\n\`\`\`${JSON.stringify(context, null, 2)}\`\`\``,
        },
      });
    }

    return this.sendMessage({
      channel,
      text: "Error Alert",
      blocks,
    });
  }

  async uploadFile(channel: string, file: Buffer, filename: string) {
    try {
      const result = await this.client.files.uploadV2({
        channels: channel,
        file,
        filename,
      });

      this.logger.log(`File uploaded to ${channel}`);
      return result;
    } catch (error) {
      this.logger.error(`Failed to upload file: ${error.message}`);
      throw error;
    }
  }

  async getUserInfo(userId: string) {
    try {
      const result = await this.client.users.info({ user: userId });
      return result.user;
    } catch (error) {
      this.logger.error(`Failed to get user info: ${error.message}`);
      throw error;
    }
  }
}
```

### 5. Webhook Handler

```typescript
// webhooks/webhook.controller.ts
import {
  Controller,
  Post,
  Body,
  Headers,
  HttpException,
  HttpStatus,
  Logger,
} from "@nestjs/common";
import { createHmac } from "crypto";
import { ConfigService } from "@nestjs/config";

@Controller("webhooks")
export class WebhookController {
  private readonly logger = new Logger(WebhookController.name);

  constructor(private readonly configService: ConfigService) {}

  @Post("stripe")
  async handleStripeWebhook(
    @Body() rawBody: Buffer,
    @Headers("stripe-signature") signature: string
  ) {
    // Verify webhook signature
    const secret = this.configService.get<string>("STRIPE_WEBHOOK_SECRET");

    if (!this.verifyStripeSignature(rawBody, signature, secret)) {
      throw new HttpException("Invalid signature", HttpStatus.UNAUTHORIZED);
    }

    const event = JSON.parse(rawBody.toString());

    this.logger.log(`Received Stripe webhook: ${event.type}`);

    // Handle different event types
    switch (event.type) {
      case "payment_intent.succeeded":
        await this.handlePaymentSuccess(event.data.object);
        break;

      case "payment_intent.payment_failed":
        await this.handlePaymentFailure(event.data.object);
        break;

      case "customer.subscription.created":
        await this.handleSubscriptionCreated(event.data.object);
        break;

      case "customer.subscription.deleted":
        await this.handleSubscriptionCancelled(event.data.object);
        break;

      default:
        this.logger.warn(`Unhandled event type: ${event.type}`);
    }

    return { received: true };
  }

  @Post("github")
  async handleGitHubWebhook(
    @Body() payload: any,
    @Headers("x-github-event") event: string,
    @Headers("x-hub-signature-256") signature: string
  ) {
    // Verify webhook signature
    const secret = this.configService.get<string>("GITHUB_WEBHOOK_SECRET");

    if (!this.verifyGitHubSignature(payload, signature, secret)) {
      throw new HttpException("Invalid signature", HttpStatus.UNAUTHORIZED);
    }

    this.logger.log(`Received GitHub webhook: ${event}`);

    // Handle different event types
    switch (event) {
      case "push":
        await this.handleGitHubPush(payload);
        break;

      case "pull_request":
        await this.handleGitHubPullRequest(payload);
        break;

      case "issues":
        await this.handleGitHubIssue(payload);
        break;

      default:
        this.logger.warn(`Unhandled GitHub event: ${event}`);
    }

    return { received: true };
  }

  @Post("slack")
  async handleSlackWebhook(@Body() payload: any) {
    // Slack URL verification
    if (payload.type === "url_verification") {
      return { challenge: payload.challenge };
    }

    // Handle Slack events
    if (payload.type === "event_callback") {
      const event = payload.event;

      this.logger.log(`Received Slack event: ${event.type}`);

      switch (event.type) {
        case "message":
          await this.handleSlackMessage(event);
          break;

        case "app_mention":
          await this.handleSlackMention(event);
          break;

        default:
          this.logger.warn(`Unhandled Slack event: ${event.type}`);
      }
    }

    return { ok: true };
  }

  // Signature verification methods
  private verifyStripeSignature(
    payload: Buffer,
    signature: string,
    secret: string
  ): boolean {
    const timestamp = signature.split(",")[0].split("=")[1];
    const sig = signature.split(",")[1].split("=")[1];

    const signedPayload = `${timestamp}.${payload.toString()}`;
    const expectedSig = createHmac("sha256", secret)
      .update(signedPayload)
      .digest("hex");

    return sig === expectedSig;
  }

  private verifyGitHubSignature(
    payload: any,
    signature: string,
    secret: string
  ): boolean {
    const expectedSig = createHmac("sha256", secret)
      .update(JSON.stringify(payload))
      .digest("hex");

    const receivedSig = signature.replace("sha256=", "");

    return receivedSig === expectedSig;
  }

  // Event handlers
  private async handlePaymentSuccess(paymentIntent: any) {
    this.logger.log(`Payment succeeded: ${paymentIntent.id}`);
    // Update order status, send confirmation email, etc.
  }

  private async handlePaymentFailure(paymentIntent: any) {
    this.logger.error(`Payment failed: ${paymentIntent.id}`);
    // Notify user, retry logic, etc.
  }

  private async handleSubscriptionCreated(subscription: any) {
    this.logger.log(`Subscription created: ${subscription.id}`);
    // Activate user features, send welcome email, etc.
  }

  private async handleSubscriptionCancelled(subscription: any) {
    this.logger.log(`Subscription cancelled: ${subscription.id}`);
    // Deactivate features, send cancellation email, etc.
  }

  private async handleGitHubPush(payload: any) {
    this.logger.log(`Push to ${payload.repository.full_name}`);
    // Trigger CI/CD, notify team, etc.
  }

  private async handleGitHubPullRequest(payload: any) {
    this.logger.log(`PR ${payload.action}: ${payload.pull_request.title}`);
    // Run tests, notify reviewers, etc.
  }

  private async handleGitHubIssue(payload: any) {
    this.logger.log(`Issue ${payload.action}: ${payload.issue.title}`);
    // Create ticket, notify team, etc.
  }

  private async handleSlackMessage(event: any) {
    this.logger.log(`Slack message from ${event.user}`);
    // Process message, trigger bot response, etc.
  }

  private async handleSlackMention(event: any) {
    this.logger.log(`Mentioned by ${event.user}`);
    // Respond to mention, trigger action, etc.
  }
}
```

### 6. API Rate Limiting & Caching

```typescript
// common/api-cache.service.ts
import { Injectable, Inject } from "@nestjs/common";
import { CACHE_MANAGER } from "@nestjs/cache-manager";
import { Cache } from "cache-manager";

@Injectable()
export class ApiCacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async getCached<T>(
    key: string,
    fetchFn: () => Promise<T>,
    ttl: number = 300 // 5 minutes default
  ): Promise<T> {
    // Try to get from cache
    const cached = await this.cacheManager.get<T>(key);

    if (cached !== undefined) {
      return cached;
    }

    // Fetch fresh data
    const data = await fetchFn();

    // Store in cache
    await this.cacheManager.set(key, data, ttl);

    return data;
  }

  async invalidate(key: string): Promise<void> {
    await this.cacheManager.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // Implementation depends on cache store
    // For Redis: use SCAN or KEYS
    const store: any = this.cacheManager.store;

    if (store.keys) {
      const keys = await store.keys(pattern);
      await Promise.all(keys.map((key) => this.cacheManager.del(key)));
    }
  }
}
```

```typescript
// Example usage with caching
@Injectable()
export class GitHubCachedService {
  constructor(
    private readonly githubService: GitHubService,
    private readonly cacheService: ApiCacheService
  ) {}

  async getUserProfile(username: string): Promise<GitHubUser> {
    return this.cacheService.getCached(
      `github:user:${username}`,
      () => this.githubService.getUserProfile(username),
      600 // Cache for 10 minutes
    );
  }
}
```

## Best Practices

### 1. **Error Handling & Retries**

```typescript
// Implement exponential backoff
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;

      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```

### 2. **Rate Limit Handling**

```typescript
// Respect rate limits
class RateLimitedClient {
  private lastRequestTime = 0;
  private minRequestInterval = 1000; // 1 second

  async request(url: string) {
    const now = Date.now();
    const timeSinceLastRequest = now - this.lastRequestTime;

    if (timeSinceLastRequest < this.minRequestInterval) {
      await new Promise((resolve) =>
        setTimeout(resolve, this.minRequestInterval - timeSinceLastRequest)
      );
    }

    this.lastRequestTime = Date.now();
    return fetch(url);
  }
}
```

### 3. **Webhook Security**

- Always verify webhook signatures
- Use HTTPS endpoints
- Implement replay attack prevention
- Rate limit webhook endpoints
- Log all webhook events

### 4. **API Key Management**

- Store API keys in environment variables
- Never commit keys to version control
- Use separate keys for development/production
- Rotate keys regularly
- Monitor API key usage

### 5. **Timeout Configuration**

```typescript
const config: AxiosRequestConfig = {
  timeout: 10000, // 10 seconds
  timeoutErrorMessage: "Request timed out",
};
```

### 6. **Response Validation**

```typescript
// Validate API responses
import { z } from "zod";

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
});

const response = await httpClient.get("/user/123");
const validated = UserSchema.parse(response.data);
```

### 7. **Circuit Breaker Pattern**

```typescript
// Prevent cascading failures
class CircuitBreaker {
  private failures = 0;
  private threshold = 5;
  private isOpen = false;
  private resetTimeout = 60000; // 1 minute

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.isOpen) {
      throw new Error("Circuit breaker is open");
    }

    try {
      const result = await fn();
      this.failures = 0;
      return result;
    } catch (error) {
      this.failures++;

      if (this.failures >= this.threshold) {
        this.isOpen = true;
        setTimeout(() => {
          this.isOpen = false;
          this.failures = 0;
        }, this.resetTimeout);
      }

      throw error;
    }
  }
}
```

## Key Takeaways

✅ **Use typed HTTP clients for type safety and better DX**  
✅ **Implement retry logic with exponential backoff**  
✅ **Always verify webhook signatures for security**  
✅ **Cache API responses to reduce external calls**  
✅ **Handle rate limits gracefully**  
✅ **Use environment variables for API credentials**  
✅ **Implement circuit breakers for resilience**  
✅ **Log all external API interactions**  
✅ **Handle timeouts and errors properly**  
✅ **Monitor API usage and costs**

Third-party integrations are essential for modern applications, enabling rapid feature development by leveraging existing platforms and services.
