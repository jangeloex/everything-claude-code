# Deployment Patterns

Patterns and best practices for deploying JavaScript/TypeScript applications.

## Docker

### Multi-stage Dockerfile for Node.js

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 nodeuser

COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist
COPY --from=builder --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/package.json ./

USER nodeuser
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### docker-compose for local dev

```yaml
version: '3.9'
services:
  app:
    build:
      context: .
      target: builder
    ports:
      - '3000:3000'
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - '5432:5432'

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'

volumes:
  postgres_data:
```

## GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
        run: |
          echo "Deploying..."
```

## Environment Configuration

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type Env = z.infer<typeof envSchema>;

function loadEnv(): Env {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error('Invalid environment variables:', result.error.flatten());
    process.exit(1);
  }
  return result.data;
}

export const env = loadEnv();
```

## Health Check Endpoint

```typescript
// src/routes/health.ts
import { Router } from 'express';
import { db } from '../db';

const router = Router();

/** Liveness probe — app is running */
router.get('/healthz', (_req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

/** Readiness probe — app can serve traffic */
router.get('/readyz', async (_req, res) => {
  try {
    await db.raw('SELECT 1');
    res.json({ status: 'ready', db: 'ok' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', db: 'error' });
  }
});

export { router as healthRouter };
```

## Graceful Shutdown

```typescript
// src/server.ts
import { createServer } from 'http';
import { app } from './app';
import { env } from './config/env';
import { db } from './db';

const server = createServer(app);

server.listen(env.PORT, () => {
  console.log(`Server running on port ${env.PORT}`);
});

function shutdown(signal: string) {
  console.log(`Received ${signal}, shutting down gracefully`);
  server.close(async () => {
    await db.destroy();
    console.log('Server closed');
    process.exit(0);
  });

  // Force exit after 10s if graceful shutdown fails
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 10_000);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

## Key Principles

- **Immutable builds**: same image runs in every environment
- **12-factor app**: config via environment variables, not files
- **Health checks**: liveness + readiness probes for orchestrators
- **Graceful shutdown**: drain connections before exiting
- **Non-root containers**: never run as root in production
- **Secret management**: never bake secrets into images; use Vault, AWS SSM, or env injection
