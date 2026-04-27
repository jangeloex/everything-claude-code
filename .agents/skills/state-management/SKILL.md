# State Management Patterns

Patterns for managing application state in React applications using Zustand and React Query.

## Zustand Store Patterns

### Basic Store

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface UserState {
  user: User | null;
  isLoading: boolean;
  error: string | null;
  setUser: (user: User | null) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  reset: () => void;
}

const initialState = {
  user: null,
  isLoading: false,
  error: null,
};

export const useUserStore = create<UserState>()(
  devtools(
    persist(
      immer((set) => ({
        ...initialState,
        setUser: (user) =>
          set((state) => {
            state.user = user;
          }),
        setLoading: (isLoading) =>
          set((state) => {
            state.isLoading = isLoading;
          }),
        setError: (error) =>
          set((state) => {
            state.error = error;
          }),
        reset: () => set(initialState),
      })),
      { name: 'user-store' }
    )
  )
);
```

### Slice Pattern (for large stores)

```typescript
import { StateCreator } from 'zustand';

interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  total: () => number;
}

export const createCartSlice: StateCreator<CartSlice> = (set, get) => ({
  items: [],
  addItem: (item) =>
    set((state) => ({
      items: [...state.items, item],
    })),
  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((i) => i.id !== id),
    })),
  clearCart: () => set({ items: [] }),
  total: () => get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),
});
```

## React Query Patterns

### Query Factory

```typescript
import { queryOptions } from '@tanstack/react-query';

// Centralized query key factory
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

export const userQueries = {
  list: (filters: UserFilters) =>
    queryOptions({
      queryKey: userKeys.list(filters),
      queryFn: () => fetchUsers(filters),
      staleTime: 1000 * 60 * 5, // 5 minutes
    }),
  detail: (id: string) =>
    queryOptions({
      queryKey: userKeys.detail(id),
      queryFn: () => fetchUser(id),
      staleTime: 1000 * 60 * 10, // 10 minutes
    }),
};
```

### Optimistic Updates

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateUserDto) => updateUser(data),
    onMutate: async (newData) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: userKeys.detail(newData.id) });

      // Snapshot previous value
      const previous = queryClient.getQueryData(userKeys.detail(newData.id));

      // Optimistically update
      queryClient.setQueryData(userKeys.detail(newData.id), (old: User) => ({
        ...old,
        ...newData,
      }));

      return { previous };
    },
    onError: (_err, newData, context) => {
      // Rollback on error
      queryClient.setQueryData(userKeys.detail(newData.id), context?.previous);
    },
    onSettled: (_data, _err, newData) => {
      queryClient.invalidateQueries({ queryKey: userKeys.detail(newData.id) });
    },
  });
}
```

## Context + Reducer (lightweight alternative)

```typescript
import { createContext, useContext, useReducer, ReactNode } from 'react';

type Action =
  | { type: 'SET_THEME'; payload: 'light' | 'dark' }
  | { type: 'TOGGLE_SIDEBAR' }
  | { type: 'SET_LOCALE'; payload: string };

interface UIState {
  theme: 'light' | 'dark';
  sidebarOpen: boolean;
  locale: string;
}

function uiReducer(state: UIState, action: Action): UIState {
  switch (action.type) {
    case 'SET_THEME':
      return { ...state, theme: action.payload };
    case 'TOGGLE_SIDEBAR':
      return { ...state, sidebarOpen: !state.sidebarOpen };
    case 'SET_LOCALE':
      return { ...state, locale: action.payload };
    default:
      return state;
  }
}

const UIContext = createContext<{
  state: UIState;
  dispatch: React.Dispatch<Action>;
} | null>(null);

export function UIProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(uiReducer, {
    theme: 'light',
    sidebarOpen: true,
    locale: 'en',
  });

  return <UIContext.Provider value={{ state, dispatch }}>{children}</UIContext.Provider>;
}

export function useUI() {
  const ctx = useContext(UIContext);
  if (!ctx) throw new Error('useUI must be used within UIProvider');
  return ctx;
}
```

## When to Use What

- **Zustand**: Global client state (auth, cart, UI preferences) that needs to persist or be shared across many components
- **React Query**: Server state — anything fetched from an API
- **Context + Reducer**: Localized state for a feature subtree, avoids prop drilling without global overhead
- **useState/useReducer**: Component-local state with no sharing needs
