# Error Handling Patterns

Standardized error handling patterns for frontend and backend, including typed errors, error boundaries, toast notifications, and API error normalization.

## Backend Error Classes

```typescript
// errors/AppError.ts
export type ErrorCode =
  | 'NOT_FOUND'
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'
  | 'VALIDATION_ERROR'
  | 'CONFLICT'
  | 'INTERNAL_ERROR'
  | 'BAD_REQUEST'
  | 'RATE_LIMITED';

export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly code: ErrorCode,
    public readonly statusCode: number,
    public readonly details?: unknown
  ) {
    super(message);
    this.name = 'AppError';
    Object.setPrototypeOf(this, AppError.prototype);
  }

  static notFound(resource: string) {
    return new AppError(`${resource} not found`, 'NOT_FOUND', 404);
  }

  static unauthorized(message = 'Unauthorized') {
    return new AppError(message, 'UNAUTHORIZED', 401);
  }

  static forbidden(message = 'Forbidden') {
    return new AppError(message, 'FORBIDDEN', 403);
  }

  static badRequest(message: string, details?: unknown) {
    return new AppError(message, 'BAD_REQUEST', 400, details);
  }

  static conflict(message: string) {
    return new AppError(message, 'CONFLICT', 409);
  }

  static internal(message = 'Internal server error') {
    return new AppError(message, 'INTERNAL_ERROR', 500);
  }

  toJSON() {
    return {
      error: {
        code: this.code,
        message: this.message,
        ...(this.details ? { details: this.details } : {}),
      },
    };
  }
}
```

## Express Error Middleware

```typescript
// middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/AppError';
import { ZodError } from 'zod';

export function errorHandler(
  err: unknown,
  req: Request,
  res: Response,
  _next: NextFunction
) {
  // Handle Zod validation errors
  if (err instanceof ZodError) {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Validation failed',
        details: err.flatten().fieldErrors,
      },
    });
  }

  // Handle known app errors
  if (err instanceof AppError) {
    return res.status(err.statusCode).json(err.toJSON());
  }

  // Log unexpected errors
  console.error('[Unhandled Error]', err);

  // Generic fallback
  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
  });
}
```

## Frontend Error Boundary

```tsx
// components/ErrorBoundary.tsx
import React, { Component, ReactNode } from 'react';

interface Props {
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  children: ReactNode;
}

interface State {
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  static getDerivedStateFromError(error: Error): State {
    return { error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('[ErrorBoundary]', error, info.componentStack);
  }

  reset = () => this.setState({ error: null });

  render() {
    const { error } = this.state;
    const { fallback, children } = this.props;

    if (error) {
      if (typeof fallback === 'function') return fallback(error, this.reset);
      if (fallback) return fallback;
      return (
        <div role="alert" style={{ padding: 16 }}>
          <h2>Something went wrong</h2>
          <pre style={{ color: 'red' }}>{error.message}</pre>
          <button onClick={this.reset}>Try again</button>
        </div>
      );
    }

    return children;
  }
}
```

## API Error Normalization (Client)

```typescript
// lib/apiClient.ts
export interface ApiErrorPayload {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

export class ApiError extends Error {
  constructor(
    public readonly payload: ApiErrorPayload,
    public readonly status: number
  ) {
    super(payload.message);
    this.name = 'ApiError';
  }

  get code() {
    return this.payload.code;
  }

  get details() {
    return this.payload.details;
  }

  isNotFound() { return this.status === 404; }
  isUnauthorized() { return this.status === 401; }
  isForbidden() { return this.status === 403; }
  isValidation() { return this.status === 400; }
}

export async function fetchApi<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, {
    headers: { 'Content-Type': 'application/json', ...init?.headers },
    ...init,
  });

  if (!res.ok) {
    let payload: ApiErrorPayload;
    try {
      const body = await res.json();
      payload = body.error ?? { code: 'UNKNOWN', message: res.statusText };
    } catch {
      payload = { code: 'UNKNOWN', message: res.statusText };
    }
    throw new ApiError(payload, res.status);
  }

  return res.json() as Promise<T>;
}
```

## Usage Notes

- Always use `AppError` factory methods on the backend for consistent HTTP status codes.
- Wrap Zod parsing in route handlers and let the error middleware handle `ZodError`.
- On the frontend, wrap page-level trees with `<ErrorBoundary>` and use `ApiError.isUnauthorized()` to trigger redirects to login.
- Pair with the `useAsync` hook (see `frontend-patterns`) to surface loading/error states in UI components.
