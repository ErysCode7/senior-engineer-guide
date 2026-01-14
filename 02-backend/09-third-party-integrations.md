# Third-Party API Integrations

## Overview

Modern applications rely on third-party APIs for payment processing, social authentication, analytics, and more. This guide covers best practices for integrating external APIs with Express, including HTTP clients, error handling, rate limiting, and testing strategies.

## HTTP Clients

### 1. Axios Setup

```bash
npm install axios
npm install -D @types/axios
```

```typescript
// src/config/http-client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';

export const createHttpClient = (baseURL: string, config?: AxiosRequestConfig): AxiosInstance => {
  const client = axios.create({
    baseURL,
    timeout: 10000,
    headers: {
      'Content-Type': 'application/json',
    },
    ...config,
  });

  // Request interceptor
  client.interceptors.request.use(
    (config) => {
      console.log(`${config.method?.toUpperCase()} ${config.url}`);
      return config;
    },
    (error) => {
      return Promise.reject(error);
    }
  );

  // Response interceptor
  client.interceptors.response.use(
    (response) => response,
    (error) => {
      if (error.response) {
        console.error('API Error:', error.response.status, error.response.data);
      } else if (error.request) {
        console.error('Network Error:', error.message);
      }
      return Promise.reject(error);
    }
  );

  return client;
};
```

## API Client Pattern

```typescript
// src/clients/github.client.ts
import { createHttpClient } from '../config/http-client';
import { AppError } from '../utils/AppError';

export class GitHubClient {
  private client;

  constructor() {
    this.client = createHttpClient('https://api.github.com', {
      headers: {
        Authorization: `token ${process.env.GITHUB_TOKEN}`,
        Accept: 'application/vnd.github.v3+json',
      },
    });
  }

  async getUser(username: string) {
    try {
      const response = await this.client.get(`/users/${username}`);
      return response.data;
    } catch (error: any) {
      if (error.response?.status === 404) {
        throw new AppError('User not found', 404);
      }
      throw new AppError('Failed to fetch user', 500);
    }
  }

  async getUserRepos(username: string) {
    try {
      const response = await this.client.get(`/users/${username}/repos`, {
        params: {
          sort: 'updated',
          per_page: 10,
        },
      });
      return response.data;
    } catch (error) {
      throw new AppError('Failed to fetch repositories', 500);
    }
  }

  async searchRepositories(query: string, page: number = 1) {
    try {
      const response = await this.client.get('/search/repositories', {
        params: {
          q: query,
          page,
          per_page: 30,
        },
      });
      return response.data;
    } catch (error) {
      throw new AppError('Search failed', 500);
    }
  }
}

export default new GitHubClient();
```

## Retry Logic

```typescript
// src/utils/retry.ts
import axios, { AxiosError, AxiosRequestConfig } from 'axios';

interface RetryConfig {
  maxRetries: number;
  retryDelay: number;
  retryCondition?: (error: AxiosError) => boolean;
}

export const retryableRequest = async <T>(
  requestFn: () => Promise<T>,
  config: RetryConfig = {
    maxRetries: 3,
    retryDelay: 1000,
  }
): Promise<T> => {
  let lastError: any;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error: any) {
      lastError = error;

      if (attempt === config.maxRetries) {
        break;
      }

      // Check if we should retry
      if (config.retryCondition && !config.retryCondition(error)) {
        break;
      }

      // Default retry condition: 5xx errors or network errors
      const shouldRetry =
        !error.response || (error.response.status >= 500 && error.response.status < 600);

      if (!shouldRetry) {
        break;
      }

      // Exponential backoff
      const delay = config.retryDelay * Math.pow(2, attempt);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
};
```

```typescript
// Usage
const data = await retryableRequest(
  () => githubClient.getUser('username'),
  { maxRetries: 3, retryDelay: 1000 }
);
```

## Rate Limiting

```typescript
// src/utils/rate-limiter.ts
export class RateLimiter {
  private requests: number[] = [];
  private maxRequests: number;
  private windowMs: number;

  constructor(maxRequests: number, windowMs: number) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  async acquire(): Promise<void> {
    const now = Date.now();
    this.requests = this.requests.filter((time) => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = this.requests[0];
      const delay = this.windowMs - (now - oldestRequest);
      await new Promise((resolve) => setTimeout(resolve, delay));
      return this.acquire();
    }

    this.requests.push(now);
  }
}

// Usage
const rateLimiter = new RateLimiter(10, 60000); // 10 requests per minute

async function makeAPICall() {
  await rateLimiter.acquire();
  return await apiClient.get('/endpoint');
}
```

## Caching

```typescript
// src/utils/cache-client.ts
import { redis } from '../config/redis';

export class CacheClient {
  async get<T>(key: string): Promise<T | null> {
    const cached = await redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }

  async set(key: string, value: any, ttlSeconds: number = 3600): Promise<void> {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
  }

  async delete(key: string): Promise<void> {
    await redis.del(key);
  }

  async wrap<T>(
    key: string,
    fetchFn: () => Promise<T>,
    ttlSeconds: number = 3600
  ): Promise<T> {
    const cached = await this.get<T>(key);
    
    if (cached !== null) {
      return cached;
    }

    const data = await fetchFn();
    await this.set(key, data, ttlSeconds);
    return data;
  }
}

export default new CacheClient();
```

```typescript
// Usage
const user = await cacheClient.wrap(
  `github:user:${username}`,
  () => githubClient.getUser(username),
  3600
);
```

## Weather API Example

```typescript
// src/clients/weather.client.ts
import { createHttpClient } from '../config/http-client';
import { AppError } from '../utils/AppError';
import cacheClient from '../utils/cache-client';

export class WeatherClient {
  private client;
  private apiKey: string;

  constructor() {
    this.apiKey = process.env.OPENWEATHER_API_KEY || '';
    this.client = createHttpClient('https://api.openweathermap.org/data/2.5');
  }

  async getCurrentWeather(city: string) {
    const cacheKey = `weather:current:${city}`;

    return cacheClient.wrap(
      cacheKey,
      async () => {
        try {
          const response = await this.client.get('/weather', {
            params: {
              q: city,
              appid: this.apiKey,
              units: 'metric',
            },
          });

          return {
            temperature: response.data.main.temp,
            description: response.data.weather[0].description,
            humidity: response.data.main.humidity,
            windSpeed: response.data.wind.speed,
          };
        } catch (error: any) {
          if (error.response?.status === 404) {
            throw new AppError('City not found', 404);
          }
          throw new AppError('Failed to fetch weather data', 500);
        }
      },
      600 // Cache for 10 minutes
    );
  }

  async getForecast(city: string) {
    try {
      const response = await this.client.get('/forecast', {
        params: {
          q: city,
          appid: this.apiKey,
          units: 'metric',
        },
      });

      return response.data.list.map((item: any) => ({
        date: item.dt_txt,
        temperature: item.main.temp,
        description: item.weather[0].description,
      }));
    } catch (error) {
      throw new AppError('Failed to fetch forecast', 500);
    }
  }
}

export default new WeatherClient();
```

## Service Layer

```typescript
// src/services/integration.service.ts
import githubClient from '../clients/github.client';
import weatherClient from '../clients/weather.client';
import { retryableRequest } from '../utils/retry';

export class IntegrationService {
  async getGitHubProfile(username: string) {
    const [user, repos] = await Promise.all([
      retryableRequest(() => githubClient.getUser(username)),
      retryableRequest(() => githubClient.getUserRepos(username)),
    ]);

    return {
      profile: user,
      topRepositories: repos.slice(0, 5),
    };
  }

  async getWeatherData(city: string) {
    const [current, forecast] = await Promise.all([
      weatherClient.getCurrentWeather(city),
      weatherClient.getForecast(city),
    ]);

    return {
      current,
      forecast: forecast.slice(0, 5),
    };
  }
}

export default new IntegrationService();
```

## Controller

```typescript
// src/controllers/integration.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import integrationService from '../services/integration.service';

export class IntegrationController {
  getGitHubProfile = asyncHandler(async (req: Request, res: Response) => {
    const { username } = req.params;

    const data = await integrationService.getGitHubProfile(username);

    res.json({
      success: true,
      data,
    });
  });

  getWeather = asyncHandler(async (req: Request, res: Response) => {
    const { city } = req.params;

    const data = await integrationService.getWeatherData(city);

    res.json({
      success: true,
      data,
    });
  });
}

export default new IntegrationController();
```

## Webhook Handling

```typescript
// src/controllers/webhook.controller.ts
import { Request, Response } from 'express';
import crypto from 'crypto';
import { AppError } from '../utils/AppError';

export class WebhookController {
  // GitHub webhook
  handleGitHubWebhook = async (req: Request, res: Response) => {
    const signature = req.headers['x-hub-signature-256'] as string;
    const secret = process.env.GITHUB_WEBHOOK_SECRET!;

    // Verify signature
    const hmac = crypto.createHmac('sha256', secret);
    const digest = `sha256=${hmac.update(JSON.stringify(req.body)).digest('hex')}`;

    if (signature !== digest) {
      throw new AppError('Invalid signature', 401);
    }

    const event = req.headers['x-github-event'];

    switch (event) {
      case 'push':
        await this.handlePushEvent(req.body);
        break;
      case 'pull_request':
        await this.handlePullRequestEvent(req.body);
        break;
      default:
        console.log(`Unhandled event: ${event}`);
    }

    res.json({ received: true });
  };

  private async handlePushEvent(payload: any) {
    console.log('Push event:', payload.repository.name);
    // Process push event
  }

  private async handlePullRequestEvent(payload: any) {
    console.log('PR event:', payload.pull_request.title);
    // Process PR event
  }
}

export default new WebhookController();
```

## Testing

```typescript
// src/__tests__/github-client.test.ts
import nock from 'nock';
import githubClient from '../clients/github.client';

describe('GitHub Client', () => {
  afterEach(() => {
    nock.cleanAll();
  });

  it('should fetch user data', async () => {
    nock('https://api.github.com')
      .get('/users/testuser')
      .reply(200, {
        login: 'testuser',
        name: 'Test User',
        public_repos: 10,
      });

    const user = await githubClient.getUser('testuser');

    expect(user.login).toBe('testuser');
    expect(user.public_repos).toBe(10);
  });

  it('should handle 404 errors', async () => {
    nock('https://api.github.com')
      .get('/users/nonexistent')
      .reply(404);

    await expect(githubClient.getUser('nonexistent')).rejects.toThrow('User not found');
  });

  it('should retry on 500 errors', async () => {
    nock('https://api.github.com')
      .get('/users/testuser')
      .reply(500)
      .get('/users/testuser')
      .reply(200, { login: 'testuser' });

    const user = await retryableRequest(() => githubClient.getUser('testuser'));

    expect(user.login).toBe('testuser');
  });
});
```

## Circuit Breaker

```typescript
// src/utils/circuit-breaker.ts
export class CircuitBreaker {
  private failureCount = 0;
  private successCount = 0;
  private nextAttempt = Date.now();
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  constructor(
    private threshold: number = 5,
    private timeout: number = 60000,
    private monitoringPeriod: number = 120000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;

    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED';
    }
  }

  private onFailure() {
    this.failureCount++;

    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }

  getState() {
    return this.state;
  }
}
```

## Best Practices

1. **Use HTTP client libraries** (Axios, Got) with interceptors
2. **Implement retry logic** for transient failures
3. **Cache responses** to reduce API calls
4. **Rate limit requests** to stay within quotas
5. **Verify webhook signatures** for security
6. **Use circuit breakers** to prevent cascading failures
7. **Log all API interactions** for debugging
8. **Handle errors gracefully** with meaningful messages
9. **Test with mocks** to avoid hitting real APIs
10. **Monitor API usage** and costs

## Key Takeaways

1. **HTTP clients** abstract API communication
2. **Retry logic** handles transient failures
3. **Caching** reduces costs and improves performance
4. **Rate limiting** prevents quota exhaustion
5. **Circuit breakers** protect against cascading failures
6. **Webhooks** enable real-time integrations
7. **Testing** ensures reliability without hitting real APIs
8. **Error handling** provides clear feedback

Well-designed integrations make your application extensible and maintainable.
