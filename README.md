<p align="center">
  <a href="http://nestjs.com/" target="blank"><img src="https://nestjs.com/img/logo-small.svg" width="120" alt="Nest Logo" /></a>
</p>

<p align="center">
    <a href='https://img.shields.io/npm/l/@sorodriguezz/nest-resilience'><img src="https://img.shields.io/npm/l/@sorodriguezz/nest-resilience" alt="MIT License" /></a>
</p>

English | [Español](./README.es.md)

**Nest Resilience** provides **resilience patterns** such as Circuit Breaker, Retry, Timeout, and Fallback for **NestJS** (and Node.js). It allows you to configure and apply these patterns easily through **interceptors**, **decorators**, and a **dynamic module**.

## 🚀 Main Features

- ✅ **Circuit Breaker**: Protects your application from repetitive failures or unstable services.
- ✅ **Retry**: Automatically retries a failed operation.
- ✅ **Timeout**: Stops operations that take too long.
- ✅ **Fallback**: Returns an alternative response when an operation fails.

## 📦 Installation

Install using NPM￼:

```
npm install @sorodriguezz/nest-resilience
```

Or with Yarn￼:

```
yarn add @sorodriguezz/nest-resilience
```

### Requirements

- NestJS (v9 or later recommended)
- Node.js 16+ (for ES2020 support)

## 📌 Basic Usage in NestJS

### 1️⃣ Import the module

In your `AppModule` (or any module where you need it), import `ResilienceModule` and configure the desired patterns:

```typescript
import { Module } from "@nestjs/common";
import { ResilienceModule } from "@sorodriguezz/nest-resilience";

@Module({
  imports: [
    ResilienceModule.forRoot({
      circuitBreaker: {
        enabled: true,
        timeout: 2000,
        errorThresholdPercentage: 50,
        resetTimeout: 10000,
      },
      retry: {
        enabled: true,
        maxRetries: 3,
        delayMs: 500,
      },
      timeout: {
        enabled: true,
        timeoutMs: 3000,
      },
      fallback: {
        enabled: true,
        fallbackMethod: () => ({ message: "Fallback result" }),
      },
    }),
  ],
})
export class AppModule {}
```

If you need to load configuration asynchronously, use `forRootAsync()`:

```typescript
ResilienceModule.forRootAsync({
  useFactory: async () => ({
    circuitBreaker: { enabled: true, timeout: 2000 },
    retry: { enabled: true, maxRetries: 5 },
  }),
});
```

### 2️⃣ Apply decorators on endpoints

You can use the provided decorators directly in your NestJS controllers:

```typescript
import { Controller, Get } from "@nestjs/common";
import {
  UseCircuitBreaker,
  UseRetry,
  UseTimeout,
  UseFallback,
} from "@sorodriguezz/nest-resilience";

@Controller("demo")
export class DemoController {
  @Get("retry")
  @UseRetry()
  getWithRetry() {
    throw new Error("Forcing error for retry");
  }

  @Get("timeout")
  @UseTimeout()
  async getWithTimeout() {
    return new Promise((resolve) =>
      setTimeout(() => resolve("Late response"), 5000)
    );
  }

  @Get("circuit")
  @UseCircuitBreaker()
  getWithCircuitBreaker() {
    if (Math.random() < 0.7) {
      throw new Error("Random Failure");
    }
    return "Success!";
  }

  @Get("fallback")
  @UseFallback()
  getWithFallback() {
    throw new Error("Forced error to use fallbackMethod");
  }
}
```

> **Note**: When a pattern is not enabled (enabled: false), the interceptor simply does nothing.

## 🧩 Usage at Service Level

You can use any pattern directly within a service, for example:

```typescript
import { Injectable } from "@nestjs/common";
import { RetryService } from "@sorodriguezz/nest-resilience";

@Injectable()
export class AppService {
  constructor(private readonly retryService: RetryService) {} // Inject the service

  async doOperationWithRetry(): Promise<string> {
    // "execute()" will retry your function if it fails
    return this.retryService.execute(async () => {
      if (Math.random() < 0.7) {
        throw new Error("Random error");
      }
      return "Success after random error!";
    });
  }
}
```

> **Note**: Using it this way gives you finer control but makes the code more repetitive. Using Decorators/Interceptors separates resilience logic from business logic (cleaner code) but with less runtime control.

## 🔗 Chaining Patterns

You can apply multiple patterns simultaneously with a single decorator (in logical order):

```typescript
@Get('all-patterns')
@UseResilienceChain() // Applies Timeout, Retry, Circuit Breaker, Fallback, etc.
myEndpoint() {
  // Endpoint logic
}
```

Or enable specific ones like this:

```typescript
@Get('timeout-retry')
@UseResilienceChain({ timeout: true, retry: true }) // Applies only Retry and Timeout
getTimeoutAndRetry() {
  return this.testService.mightFailRandomly();
}
```

## 📡 Logging

You can log the initial configuration at startup:

```typescript
imports: [
  ResilienceModule.forRoot({
    logOnStartup: true, // Logs when the app starts
    circuitBreaker: {
      enabled: true,
      timeout: 2000,
      errorThresholdPercentage: 50,
      resetTimeout: 3000,
    },
  }),
],
```

## 🔍 View Configuration

You can inspect configurations of a pattern through the service like this:

```typescript
Module({
  imports: [
    ResilienceModule.forRoot({
      fallback: {
        enabled: true,
        fallbackMethod: sayHello,
      },
    }),
  ],
  providers: [ResilienceService],
});
```

Then inject and use it inside your application:

```typescript
import { Injectable } from "@nestjs/common";
import { ResilienceService } from "@sorodriguezz/nest-resilience";

export class AppService {
  private attemptCount = 0;

  constructor(private readonly resilienceService: ResilienceService) {}

  alwaysFails() {
    console.log(this.rs.getCircuitBreakerOptions());
    throw new Error("I always fail!");
  }

  /** output:
  {
    enabled: true,
    errorThresholdPercentage: 50,
    resetTimeout: 5000,
    timeout: 1000
  }
  **/
}
```

## 📌 Basic Usage in NodeJS with Express

1️⃣ Import **nest-resilience** in your project and configure the patterns you want:

```javascript
const {
  RetryService,
  FallbackService,
} = require("@sorodriguezz/nest-resilience");

const retryService = new RetryService({
  enabled: true,
  maxRetries: 3,
  delayMs: 500,
});

const fallbackService = new FallbackService({
  enabled: true,
  fallbackMethod: () => ({ message: "Fallback used!" }),
});
```

## 2️⃣ Apply them in your endpoints:

```javascript
app.get("/test/fallback", (req, res) => {
  try {
    throw new Error("Something failed");
  } catch (err) {
    const fallbackValue = fallbackService.executeFallback();
    res.json({ data: fallbackValue });
  }
});

app.get("/test/retry", async (req, res) => {
  try {
    const result = await retryService.execute(() => {
      if (Math.random() < 0.7) throw new Error("Random fail");
      return "Success after retry!";
    });
    res.json({ data: result });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

## 📜 License

This project is distributed under the MIT License.
You can use it freely in both personal and commercial environments.
