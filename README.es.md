<p align="center">
  <a href="http://nestjs.com/" target="blank"><img src="https://nestjs.com/img/logo-small.svg" width="120" alt="Nest Logo" /></a>
</p>

<p align="center">
    <a href='https://img.shields.io/npm/l/@sorodriguezz/nest-resilience'><img src="https://img.shields.io/npm/l/@sorodriguezz/nest-resilience" alt="MIT License" /></a>
</p>

**Español** | [English](./README.md)

**Nest Resilience** proporciona **patrones de resiliencia** como Circuit Breaker, Retry, Timeout y Fallback para **NestJS** (y Node.js). Permite configurar y aplicar estos patrones de forma sencilla mediante **interceptores**, **decoradores** y un **módulo dinámico**.

## 🚀 Características principales

✅ **Circuit Breaker**: Protege tu aplicación de fallos repetitivos o servicios inestables.  
✅ **Retry**: Reintenta automáticamente una operación fallida.  
✅ **Timeout**: Detiene las operaciones que toman demasiado tiempo.  
✅ **Fallback**: Devuelve una respuesta alternativa cuando una operación falla.

## 📦 Instalación

Puedes realizar la instalación con [NPM](https://www.npmjs.com/):

```bash
npm install @sorodriguezz/nest-resilience
```

O usando [Yarn](https://yarnpkg.com/):

```bash
yarn add @sorodriguezz/nest-resilience
```

### Requisitos

- NestJS (v9 o superior recomendado).
- Node.js 16+ (para soporte de ES2020).

## 📌 Uso básico en NestJS

### 1️⃣ Importar el módulo

En tu `AppModule` (o el módulo donde lo necesites), importa `ResilienceModule` y configura los patrones deseados:

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

Si necesitas cargar la configuración de forma asíncrona, usa `forRootAsync()`:

```typescript
ResilienceModule.forRootAsync({
  useFactory: async () => ({
    circuitBreaker: { enabled: true, timeout: 2000 },
    retry: { enabled: true, maxRetries: 5 },
  }),
});
```

---

### 2️⃣ Aplicar decoradores en los endpoints

Puedes usar los decoradores que provee la librería en tus controladores de NestJS:

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
    throw new Error("Forzando error para reintento");
  }

  @Get("timeout")
  @UseTimeout()
  async getWithTimeout() {
    return new Promise((resolve) =>
      setTimeout(() => resolve("Respuesta tardía"), 5000)
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
    throw new Error("Forzamos error para usar fallbackMethod");
  }
}
```

> **Nota**: Cuando un patrón no está habilitado (`enabled: false`), el interceptor simplemente no hace nada.

## Uso a nivel de servicios

Puedes usar el patron que quieras a nivel de servicio como:

```typescript
import { Injectable } from "@nestjs/common";
import { RetryService } from "@sorodriguezz/nest-resilience";

@Injectable()
export class AppService {
  constructor(private readonly retryService: RetryService) {} // Inyectamos el servicio

  async doOperationWithRetry(): Promise<string> {
    // "execute()" reintentará tu función si falla
    return this.retryService.execute(async () => {
      // Lógica que podría fallar
      if (Math.random() < 0.7) {
        throw new Error("Random error");
      }
      return "Success after random error!";
    });
  }
}
```

> **Nota:** Al usarlo de esta manera se vuelve más repetitivo pero se tiene un control más exacto. En cambio por Decoradorador/Interceptor separa la lógica de los patrones con la lógica de negocio (volviendo el código mas limpio). Pero se tiene menos control, ya que es en tiempo de Request.

## 🔗 Uso en cadena

Puedes aplicar múltiples patrones, todos a la vez y simultáneamente con un solo decorador (Aplicandolos en orden lógico):

```typescript
@Get('all-patterns')
@UseResilienceChain() // Aplica Timeout, Retry, Circuit Breaker, Fallback, etc.
myEndpoint() {
  // Lógica del endpoint
}
```

O puedes habilitar los que desees usar de la siguiente manera:

```typescript
  @Get('timeout-retry')
  @UseResilienceChain({ timeout: true, retry: true }) // Solo aplica Retry y TimeOut
  getTimeoutAndRetry() {
    return this.testService.mightFailRandomly();
  }
```

## 📡 Logs

También puedes loggear la configuración inicial al arrancar la aplicación:

```typescript
  imports: [
    ResilienceModule.forRoot({
      logOnStartup: true, // Para loggear apenas arranca
      circuitBreaker: {
        enabled: true,
        timeout: 2000,
        errorThresholdPercentage: 50,
        resetTimeout: 3000,
      },
    }),
  ],
```

## 🔍 Ver configuraciones

Puedes ver las configuraciones según un patrón a través del servicio, de la siguiente manera:

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
  providers: [ResilienceService], // Proveer el servicio
});
```

Para utilizarlo dentro de la aplicación, se inyecta en el constructor y se utiliza de la siguiente manera:

```typescript
import { Injectable } from '@nestjs/common';
import { ResilienceService } from '@sorodriguezz/nest-resilience';

export class AppService {
  private attemptCount = 0;

  constructor(private readonly resilienceService: ResilienceService) {}

  alwaysFails() {
    console.log(this.rs.getCircuitBreakerOptions()); // Obtiene en fallback
    throw new Error('I always fail!');
  }

  /** output:
    {
      enabled: true,
      errorThresholdPercentage: 50,
      resetTimeout: 5000,
      timeout: 1000
    }
   **/
```

## 📌 Uso básico en NodeJS con Express

1️⃣ Importa **nest resilience** en tu proyecto usando el patron que quieras y con la configuración que quieras:

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
  fallbackMethod: () => ({ message: "Fallback used2!" }),
});
```

2️⃣ Aplica en tus EndPoints, para usar solo debes ejecutar los métodos a traves de sus instancias:

```javascript
app.get("/test/fallback", (req, res) => {
  try {
    // Forzamos error
    throw new Error("Something failed");
  } catch (err) {
    // Llamamos a fallback
    const fallbackValue = fallbackService.executeFallback();
    res.json({ data: fallbackValue });
  }
});

app.get("/test/retry", async (req, res) => {
  try {
    const result = await retryService.execute(() => {
      // Lógica que falla
      if (Math.random() < 0.7) throw new Error("Random fail");
      return "Success after retry!";
    });
    res.json({ data: result });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

## 📜 Licencia

Este proyecto se distribuye bajo la licencia **MIT**. Puedes usarlo libremente en entornos personales y comerciales.
