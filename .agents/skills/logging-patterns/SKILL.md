# Logging Patterns

Structured logging utilities for Node.js/TypeScript applications with support for multiple transports, log levels, and context propagation.

## Core Logger

```typescript
import { createLogger, format, transports, Logger } from 'winston';

const { combine, timestamp, errors, json, colorize, simple } = format;

const isDev = process.env.NODE_ENV !== 'production';

export const logger: Logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
    errors({ stack: true }),
    json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'app',
    version: process.env.npm_package_version,
  },
  transports: [
    isDev
      ? new transports.Console({ format: combine(colorize(), simple()) })
      : new transports.Console(),
  ],
});
```

## Child Logger with Context

```typescript
// Attach request-scoped metadata to every log in a handler
export function childLogger(meta: Record<string, unknown>): Logger {
  return logger.child(meta);
}

// Example usage in Express middleware
export function requestLogger(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const log = childLogger({
    requestId: req.headers['x-request-id'] ?? crypto.randomUUID(),
    method: req.method,
    path: req.path,
    ip: req.ip,
  });

  // Attach to request so downstream handlers can use it
  (req as any).log = log;

  const start = Date.now();
  res.on('finish', () => {
    log.info('request completed', {
      statusCode: res.statusCode,
      durationMs: Date.now() - start,
    });
  });

  next();
}
```

## Async Context Propagation (Node 18+)

```typescript
import { AsyncLocalStorage } from 'async_hooks';

interface LogContext {
  requestId?: string;
  userId?: string;
  traceId?: string;
  [key: string]: unknown;
}

const asyncContext = new AsyncLocalStorage<LogContext>();

export function runWithContext<T>(ctx: LogContext, fn: () => T): T {
  return asyncContext.run(ctx, fn);
}

export function getContextLogger(): Logger {
  const ctx = asyncContext.getStore() ?? {};
  return logger.child(ctx);
}

// Usage: anywhere in the call stack, no need to thread `log` through params
export const log = {
  info: (msg: string, meta?: object) => getContextLogger().info(msg, meta),
  warn: (msg: string, meta?: object) => getContextLogger().warn(msg, meta),
  error: (msg: string, meta?: object) => getContextLogger().error(msg, meta),
  debug: (msg: string, meta?: object) => getContextLogger().debug(msg, meta),
};
```

## Error Logging Helper

```typescript
export function logError(
  err: unknown,
  context?: Record<string, unknown>
): void {
  const normalised =
    err instanceof Error ? err : new Error(String(err));

  getContextLogger().error(normalised.message, {
    ...context,
    stack: normalised.stack,
    name: normalised.name,
    // Capture extra fields from custom error classes (see error-handling skill)
    ...(err instanceof Error && 'code' in err ? { code: (err as any).code } : {}),
    ...(err instanceof Error && 'statusCode' in err
      ? { statusCode: (err as any).statusCode }
      : {}),
  });
}
```

## Performance Timing

```typescript
export function timed<T>(
  label: string,
  fn: () => Promise<T>,
  meta?: Record<string, unknown>
): Promise<T> {
  const start = performance.now();
  return fn().then(
    (result) => {
      log.debug(`${label} completed`, {
        ...meta,
        durationMs: +(performance.now() - start).toFixed(2),
      });
      return result;
    },
    (err) => {
      log.warn(`${label} failed`, {
        ...meta,
        durationMs: +(performance.now() - start).toFixed(2),
      });
      throw err;
    }
  );
}
```

## Notes

- Use `childLogger` / `runWithContext` to avoid passing a logger instance through every function signature.
- In production, ship JSON logs to your aggregator (Datadog, Loki, CloudWatch, etc.).
- Pair with the **error-handling** skill — `logError` understands `ApiError`'s `statusCode` and `code` fields.
- Pair with the **performance-patterns** skill — wrap expensive DB/cache calls with `timed()`.
- Never log secrets; scrub tokens/passwords before passing objects to the logger.
