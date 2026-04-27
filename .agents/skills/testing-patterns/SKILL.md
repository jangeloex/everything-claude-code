# Testing Patterns Skill

This skill provides comprehensive testing patterns for JavaScript/TypeScript projects, covering unit tests, integration tests, and component tests.

## Patterns

### Unit Testing with Vitest

```typescript
// utils/formatters.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { formatCurrency, formatDate, truncateText } from './formatters';

describe('formatCurrency', () => {
  it('formats USD by default', () => {
    expect(formatCurrency(1234.56)).toBe('$1,234.56');
  });

  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });

  it('supports other currencies', () => {
    expect(formatCurrency(100, 'EUR')).toBe('€100.00');
  });

  it('handles negative values', () => {
    expect(formatCurrency(-50)).toBe('-$50.00');
  });
});

describe('truncateText', () => {
  it('truncates text longer than limit', () => {
    expect(truncateText('Hello World', 5)).toBe('Hello...');
  });

  it('returns original text if within limit', () => {
    expect(truncateText('Hi', 10)).toBe('Hi');
  });
});
```

### Mocking Modules and APIs

```typescript
// services/userService.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { fetchUser, createUser } from './userService';
import { apiClient } from '../lib/apiClient';

vi.mock('../lib/apiClient');

const mockApiClient = vi.mocked(apiClient);

describe('userService', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  describe('fetchUser', () => {
    it('returns user data on success', async () => {
      const mockUser = { id: '1', name: 'Alice', email: 'alice@example.com' };
      mockApiClient.get.mockResolvedValueOnce({ data: mockUser });

      const user = await fetchUser('1');
      expect(user).toEqual(mockUser);
      expect(mockApiClient.get).toHaveBeenCalledWith('/users/1');
    });

    it('throws on 404', async () => {
      mockApiClient.get.mockRejectedValueOnce(new Error('Not Found'));
      await expect(fetchUser('999')).rejects.toThrow('Not Found');
    });
  });

  describe('createUser', () => {
    it('posts user data and returns created user', async () => {
      const input = { name: 'Bob', email: 'bob@example.com' };
      const created = { id: '2', ...input };
      mockApiClient.post.mockResolvedValueOnce({ data: created });

      const result = await createUser(input);
      expect(result).toEqual(created);
      expect(mockApiClient.post).toHaveBeenCalledWith('/users', input);
    });
  });
});
```

### React Component Testing with Testing Library

```typescript
// components/UserCard.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UserCard } from './UserCard';

const mockUser = {
  id: '1',
  name: 'Alice Johnson',
  email: 'alice@example.com',
  role: 'admin' as const,
};

describe('UserCard', () => {
  it('renders user information', () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText('Alice Johnson')).toBeInTheDocument();
    expect(screen.getByText('alice@example.com')).toBeInTheDocument();
    expect(screen.getByText('admin')).toBeInTheDocument();
  });

  it('calls onEdit when edit button clicked', async () => {
    const onEdit = vi.fn();
    render(<UserCard user={mockUser} onEdit={onEdit} />);

    await userEvent.click(screen.getByRole('button', { name: /edit/i }));
    expect(onEdit).toHaveBeenCalledWith(mockUser);
  });

  it('shows confirmation before delete', async () => {
    const onDelete = vi.fn();
    render(<UserCard user={mockUser} onDelete={onDelete} />);

    await userEvent.click(screen.getByRole('button', { name: /delete/i }));
    expect(screen.getByText(/are you sure/i)).toBeInTheDocument();

    await userEvent.click(screen.getByRole('button', { name: /confirm/i }));
    expect(onDelete).toHaveBeenCalledWith('1');
  });

  it('does not render action buttons without handlers', () => {
    render(<UserCard user={mockUser} />);
    expect(screen.queryByRole('button', { name: /edit/i })).not.toBeInTheDocument();
  });
});
```

### Custom Hook Testing

```typescript
// hooks/useDebounce.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { useDebounce } from './useDebounce';

describe('useDebounce', () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 300));
    expect(result.current).toBe('hello');
  });

  it('debounces value updates', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 300),
      { initialProps: { value: 'hello' } }
    );

    rerender({ value: 'world' });
    expect(result.current).toBe('hello');

    act(() => vi.advanceTimersByTime(300));
    expect(result.current).toBe('world');
  });
});
```

### Integration Test Pattern

```typescript
// features/auth/login.integration.test.ts
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { LoginForm } from './LoginForm';
import { AppProviders } from '../../test/AppProviders';

const server = setupServer(
  http.post('/api/auth/login', async ({ request }) => {
    const body = await request.json() as { email: string; password: string };
    if (body.email === 'admin@example.com' && body.password === 'correct') {
      return HttpResponse.json({ token: 'mock-jwt-token', user: { id: '1', email: body.email } });
    }
    return HttpResponse.json({ error: 'Invalid credentials' }, { status: 401 });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('Login Integration', () => {
  it('logs in with valid credentials', async () => {
    render(<LoginForm />, { wrapper: AppProviders });

    await userEvent.type(screen.getByLabelText(/email/i), 'admin@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'correct');
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByText(/welcome/i)).toBeInTheDocument();
    });
  });

  it('shows error on invalid credentials', async () => {
    render(<LoginForm />, { wrapper: AppProviders });

    await userEvent.type(screen.getByLabelText(/email/i), 'wrong@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'bad');
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByText(/invalid credentials/i)).toBeInTheDocument();
    });
  });
});
```

## Best Practices

- **Arrange-Act-Assert**: Structure each test with clear setup, action, and assertion phases
- **Test behavior, not implementation**: Focus on what the code does, not how it does it
- **One assertion per test**: Keep tests focused and failure messages clear
- **Use `userEvent` over `fireEvent`**: More realistic user interaction simulation
- **MSW for API mocking**: Intercept at the network level for integration tests
- **`vi.clearAllMocks()` in `beforeEach`**: Prevent test pollution
- **Descriptive test names**: `it('shows error message when email is invalid')` not `it('works')`
