# Frontend Patterns Skill

This skill provides reusable frontend patterns for React applications, including hooks, components, and state management patterns.

## Patterns

### Custom Hooks

#### useAsync
Manages async operations with loading, error, and data states.

```typescript
import { useState, useCallback } from 'react';

interface AsyncState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useAsync<T>(
  asyncFn: (...args: any[]) => Promise<T>
) {
  const [state, setState] = useState<AsyncState<T>>({
    data: null,
    loading: false,
    error: null,
  });

  const execute = useCallback(
    async (...args: any[]) => {
      setState({ data: null, loading: true, error: null });
      try {
        const data = await asyncFn(...args);
        setState({ data, loading: false, error: null });
        return data;
      } catch (error) {
        setState({ data: null, loading: false, error: error as Error });
        throw error;
      }
    },
    [asyncFn]
  );

  return { ...state, execute };
}
```

#### useLocalStorage
Persists state to localStorage with JSON serialization.

```typescript
import { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('useLocalStorage setValue error:', error);
    }
  };

  return [storedValue, setValue] as const;
}
```

#### useDebounce
Debounces a value to reduce unnecessary re-renders or API calls.

```typescript
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

### Component Patterns

#### Compound Components
Allows flexible composition of related components.

```typescript
import React, { createContext, useContext, useState } from 'react';

interface AccordionContextType {
  activeItem: string | null;
  setActiveItem: (id: string | null) => void;
}

const AccordionContext = createContext<AccordionContextType | null>(null);

function Accordion({ children }: { children: React.ReactNode }) {
  const [activeItem, setActiveItem] = useState<string | null>(null);
  return (
    <AccordionContext.Provider value={{ activeItem, setActiveItem }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, title, children }: {
  id: string;
  title: string;
  children: React.ReactNode;
}) {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('AccordionItem must be inside Accordion');
  const isOpen = ctx.activeItem === id;

  return (
    <div className="accordion-item">
      <button onClick={() => ctx.setActiveItem(isOpen ? null : id)}>
        {title}
      </button>
      {isOpen && <div className="accordion-content">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;
```

#### Render Props
Shares stateful logic without changing component hierarchy.

```typescript
interface RenderPropsMouseTracker {
  render: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ render }: RenderPropsMouseTracker) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e: React.MouseEvent) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return (
    <div style={{ height: '100%' }} onMouseMove={handleMouseMove}>
      {render(position)}
    </div>
  );
}
```

### State Management

#### Context + Reducer Pattern
Scalable state management without external libraries.

```typescript
import React, { createContext, useContext, useReducer } from 'react';

type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' };

interface State {
  count: number;
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    case 'RESET':
      return { count: 0 };
    default:
      return state;
  }
}

const CounterContext = createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
} | null>(null);

function CounterProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <CounterContext.Provider value={{ state, dispatch }}>
      {children}
    </CounterContext.Provider>
  );
}

function useCounter() {
  const ctx = useContext(CounterContext);
  if (!ctx) throw new Error('useCounter must be inside CounterProvider');
  return ctx;
}
```

## Usage Guidelines

- Prefer custom hooks for reusable stateful logic
- Use compound components for flexible UI composition
- Reach for context + reducer before adding Redux or Zustand
- Always type your props and state with TypeScript
- Memoize expensive computations with `useMemo` and callbacks with `useCallback`
- Co-locate state as close to where it's used as possible
