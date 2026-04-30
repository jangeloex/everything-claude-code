# Monitoring Patterns

Skill for implementing application monitoring, health checks, and metrics collection.

## Health Check Endpoint

```typescript
import { Router, Request, Response } from 'express';
import { db } from '../db';
import { redis } from '../cache';

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: Record<string, CheckResult>;
}

interface CheckResult {
  status: 'pass' | 'fail' | 'warn';
  responseTime?: number;
  message?: string;
}

async function checkDatabase(): Promise<CheckResult> {
  const start = Date.now();
  try {
    await db.raw('SELECT 1');
    return { status: 'pass', responseTime: Date.now() - start };
  } catch (err) {
    return { status: 'fail', message: (err as Error).message };
  }
}

async function checkRedis(): Promise<CheckResult> {
  const start = Date.now();
  try {
    await redis.ping();
    return { status: 'pass', responseTime: Date.now() - start };
  } catch (err) {
    return { status: 'fail', message: (err as Error).message };
  }
}

export const healthRouter = Router();

healthRouter.get('/health', async (req: Request, res: Response) => {
  const checks: Record<string, CheckResult> = {
    database: await checkDatabase(),
    redis: await checkRedis(),
  };

  const allPassing = Object.values(checks).every(c => c.status === 'pass');
  const anyFailing = Object.values(checks).some(c => c.status === 'fail');

  const status: HealthStatus['status'] = anyFailing
    ? 'unhealthy'
    : allPassing
    ? 'healthy'
    : 'degraded';

  const body: HealthStatus = {
    status,
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION ?? 'unknown',
    uptime: process.uptime(),
    checks,
  };

  res.status(status === 'unhealthy' ? 503 : 200).json(body);
});

// Lightweight liveness probe — no dependency checks
healthRouter.get('/health/live', (_req: Request, res: Response) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});
```

## Prometheus Metrics

```typescript
import client from 'prom-client';
import { Request, Response, NextFunction } from 'express';

// Enable default Node.js metrics (event loop lag, memory, etc.)
client.collectDefaultMetrics({ prefix: 'app_' });

export const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

export const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

export const activeConnections = new client.Gauge({
  name: 'http_active_connections',
  help: 'Number of active HTTP connections',
});

/** Express middleware to record request metrics */
export function metricsMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const end = httpRequestDuration.startTimer();
  activeConnections.inc();

  res.on('finish', () => {
    const labels = {
      method: req.method,
      route: req.route?.path ?? req.path,
      status_code: String(res.statusCode),
    };
    end(labels);
    httpRequestTotal.inc(labels);
    activeConnections.dec();
  });

  next();
}

/** Expose /metrics endpoint for Prometheus scraping */
export async function metricsHandler(
  _req: Request,
  res: Response
): Promise<void> {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
}
```

## Usage

```typescript
import express from 'express';
import { healthRouter } from './health';
import { metricsMiddleware, metricsHandler } from './metrics';

const app = express();

// Apply metrics tracking to all routes
app.use(metricsMiddleware);

// Health check routes (no auth required)
app.use(healthRouter);

// Prometheus scrape endpoint (restrict in prod with IP allowlist or auth)
app.get('/metrics', metricsHandler);
```

## Notes

- Keep `/health/live` dependency-free — used by Kubernetes liveness probes
- Use `/health` for readiness probes; it checks real dependencies
- Restrict `/metrics` to internal networks or add bearer token auth in production
- Add custom business metrics (e.g. `orders_processed_total`) alongside defaults
