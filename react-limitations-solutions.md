# React Limitations, Challenges & How to Overcome Them

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [React Limitations & Solutions](#react-limitations--solutions)
2. [React Anti-Patterns & How to Fix Them](#react-anti-patterns--how-to-fix-them)
3. [React vs Alternatives](#react-vs-alternatives)

---

## React Limitations & Solutions

### Performance with large lists — how do you solve it?

**Problem:** Rendering thousands of DOM nodes causes layout thrashing, janky scrolling, and high memory usage.

**Solution: Virtualization** — only render what's visible in the viewport.

```bash
npm install @tanstack/react-virtual
# or
npm install react-window
```

```tsx
// react-window — fixed-size list
import { FixedSizeList as List } from 'react-window';

const Row = ({ index, style }) => (
  <div style={style}>Row {index}</div>
);

const VirtualList = ({ items }) => (
  <List
    height={600}       // container height
    itemCount={10000}  // total items
    itemSize={50}      // each row height
    width="100%"
  >
    {Row}
  </List>
);
```

```tsx
// TanStack Virtual — more flexible, works with dynamic heights
import { useVirtualizer } from '@tanstack/react-virtual';

const VirtualList = ({ items }) => {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
};
```

**When to virtualize:** Lists with more than ~100–200 items.

---

### How do you prevent unnecessary re-renders in React?

**Root causes of unnecessary re-renders:**
1. Parent component re-renders → all children re-render by default
2. Context value changes → all consumers re-render
3. New object/array references created on every render

**Solutions:**

```tsx
// 1. React.memo — skip re-render if props haven't changed
const ExpensiveComponent = React.memo(({ data, onItemClick }) => {
  console.log('Rendering ExpensiveComponent');
  return <div>{data.map(item => <Item key={item.id} {...item} onClick={onItemClick} />)}</div>;
});

// 2. useCallback — stable function reference
const Parent = () => {
  const [count, setCount] = useState(0);
  
  // Without useCallback, new function created every render → breaks memo
  const handleItemClick = useCallback((id) => {
    console.log('Clicked:', id);
  }, []); // stable reference
  
  return <ExpensiveComponent data={items} onItemClick={handleItemClick} />;
};

// 3. useMemo — memoize expensive computed values
const ProcessedData = ({ rawData, filter }) => {
  const filteredData = useMemo(
    () => rawData.filter(item => item.type === filter).sort((a, b) => a.name.localeCompare(b.name)),
    [rawData, filter]
  );
  
  return <List items={filteredData} />;
};

// 4. State colocation — move state down to where it's needed
// BAD: State in parent causes parent + all children to re-render
const Parent = () => {
  const [inputValue, setInputValue] = useState('');
  return (
    <>
      <HeavyComponent /> {/* re-renders on every keystroke! */}
      <input value={inputValue} onChange={e => setInputValue(e.target.value)} />
    </>
  );
};

// GOOD: Colocate state in its own component
const SearchInput = () => {
  const [inputValue, setInputValue] = useState('');
  return <input value={inputValue} onChange={e => setInputValue(e.target.value)} />;
};

const Parent = () => (
  <>
    <HeavyComponent /> {/* no longer re-renders on keystrokes */}
    <SearchInput />
  </>
);
```

**Profiling re-renders:** Use React DevTools Profiler or `why-did-you-render` library.

---

### How do you solve Prop Drilling in React?

**Problem:** Passing props many levels deep through components that don't use the data themselves.

**Solutions:**

```tsx
// 1. Context API — for global/semi-global state
const UserContext = createContext(null);

const UserProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
};

// Consume anywhere in the tree
const Avatar = () => {
  const { user } = useContext(UserContext);
  return <img src={user?.avatar} alt={user?.name} />;
};
```

```tsx
// 2. Component Composition — eliminate drilling via children/render props
// BAD: prop drilling
const Page = ({ user }) => <Layout user={user}><Content user={user} /></Layout>;
const Layout = ({ user, children }) => <div><Header user={user} />{children}</div>;

// GOOD: composition
const Page = ({ user }) => (
  <Layout header={<Header user={user} />}>
    <Content user={user} />
  </Layout>
);
const Layout = ({ header, children }) => <div>{header}{children}</div>;
```

```tsx
// 3. Zustand — simple external state, no provider needed
import { create } from 'zustand';

const useUserStore = create((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));

// Use in any component without drilling
const Avatar = () => {
  const user = useUserStore((state) => state.user);
  return <img src={user?.avatar} />;
};
```

---

### How do you handle SEO challenges with React's Client-Side Rendering?

**Problem:** CSR apps send an empty HTML shell to the browser. Crawlers may not execute JavaScript, leading to poor SEO.

**Solutions:**

```tsx
// 1. Next.js SSR — render on server per request
// app/page.tsx (Next.js 13+ App Router)
async function ProductPage({ params }) {
  const product = await fetch(`/api/products/${params.id}`).then(r => r.json());
  
  return (
    <>
      <Head>
        <title>{product.name}</title>
        <meta name="description" content={product.description} />
      </Head>
      <ProductDetail product={product} />
    </>
  );
}
```

```tsx
// 2. Next.js SSG — generate HTML at build time
// pages/blog/[slug].tsx (Next.js Pages Router)
export async function getStaticPaths() {
  const posts = await fetchAllPosts();
  return { paths: posts.map(p => ({ params: { slug: p.slug } })), fallback: 'blocking' };
}

export async function getStaticProps({ params }) {
  const post = await fetchPost(params.slug);
  return { props: { post }, revalidate: 60 }; // ISR: revalidate every 60 seconds
}
```

```tsx
// 3. React Server Components (RSC) — render on server, no JS for static parts
// app/product/[id]/page.tsx
export default async function ProductPage({ params }) {
  const product = await db.products.findById(params.id); // direct DB access!
  
  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} /> {/* client component */}
    </article>
  );
}
```

**Comparison:**

| Approach | SEO | Build Time | Runtime Cost | Use Case |
|---|---|---|---|---|
| CSR | Poor | Fast | Low server | Dashboards, apps behind login |
| SSR | Excellent | Fast | High server | E-commerce, news |
| SSG | Excellent | Slow | None | Blogs, marketing |
| ISR | Excellent | Medium | Low | Content that changes infrequently |
| RSC | Excellent | Fast | Medium | Mixed content |

---

### How do you reduce large bundle sizes in React apps?

**Problem:** Large JavaScript bundles lead to slow Time-to-Interactive.

```tsx
// 1. Code splitting with React.lazy + Suspense
const Dashboard = React.lazy(() => import('./Dashboard'));
const Settings = React.lazy(() => import('./Settings'));

const App = () => (
  <Suspense fallback={<Spinner />}>
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  </Suspense>
);
```

```tsx
// 2. Dynamic imports for heavy libraries
// BAD: Import at top level
import { Chart } from 'chart.js';

// GOOD: Import dynamically when needed
const renderChart = async (data) => {
  const { Chart } = await import('chart.js');
  new Chart(canvasRef.current, { data });
};
```

```tsx
// 3. Next.js dynamic imports with SSR disabled
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./HeavyChart'), {
  ssr: false,
  loading: () => <ChartSkeleton />,
});
```

```bash
# Analyze bundle
npm install --save-dev webpack-bundle-analyzer

# In vite
npm install --save-dev rollup-plugin-visualizer
```

```js
// vite.config.ts - analyze bundle
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  plugins: [visualizer({ open: true })]
};
```

---

### How do you manage complex state in React?

```tsx
// 1. useReducer for complex local state
const initialState = { loading: false, data: null, error: null };

function fetchReducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':  return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS': return { loading: false, data: action.payload, error: null };
    case 'FETCH_ERROR':  return { loading: false, data: null, error: action.error };
    default: return state;
  }
}

const DataFetcher = ({ url }) => {
  const [state, dispatch] = useReducer(fetchReducer, initialState);
  
  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    fetch(url)
      .then(r => r.json())
      .then(data => dispatch({ type: 'FETCH_SUCCESS', payload: data }))
      .catch(error => dispatch({ type: 'FETCH_ERROR', error }));
  }, [url]);
  
  // ...
};
```

```tsx
// 2. Zustand — simple, performant global state
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useStore = create(
  devtools(
    persist(
      (set, get) => ({
        cart: [],
        addItem: (item) => set(state => ({ cart: [...state.cart, item] })),
        removeItem: (id) => set(state => ({ cart: state.cart.filter(i => i.id !== id) })),
        total: () => get().cart.reduce((sum, item) => sum + item.price, 0),
      }),
      { name: 'cart-storage' }
    )
  )
);
```

**When to use what:**

| Scenario | Solution |
|---|---|
| Simple local UI state | `useState` |
| Complex local state with transitions | `useReducer` |
| Shared UI state across components | `Context + useState/useReducer` |
| Global client state | Zustand, Jotai |
| Server/async state | TanStack Query, SWR |
| Complex global state with DevTools | Redux Toolkit |

---

### How do you prevent Memory Leaks in React?

**Common causes:**
1. Async operations that update state after component unmounts
2. Event listeners not cleaned up
3. Timers (setTimeout, setInterval) not cleared
4. Subscriptions not unsubscribed

```tsx
// 1. Cleanup async operations
useEffect(() => {
  const controller = new AbortController();
  
  fetch('/api/data', { signal: controller.signal })
    .then(r => r.json())
    .then(data => setData(data))
    .catch(err => {
      if (err.name !== 'AbortError') console.error(err);
    });
  
  return () => controller.abort(); // cleanup on unmount
}, []);
```

```tsx
// 2. Cleanup event listeners
useEffect(() => {
  const handleResize = () => setWindowWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

```tsx
// 3. Cleanup timers
useEffect(() => {
  const intervalId = setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  return () => clearInterval(intervalId);
}, []);
```

```tsx
// 4. Cleanup subscriptions (e.g., RxJS, WebSocket)
useEffect(() => {
  const subscription = dataStream$.subscribe(setData);
  return () => subscription.unsubscribe();
}, []);
```

---

### How do you reduce boilerplate code in React?

```tsx
// 1. Custom hooks — extract reusable logic
const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    fetch(url, { signal: controller.signal })
      .then(r => r.json())
      .then(data => { setData(data); setLoading(false); })
      .catch(err => { if (err.name !== 'AbortError') setError(err); });
    return () => controller.abort();
  }, [url]);
  
  return { data, loading, error };
};

// Usage
const Products = () => {
  const { data, loading, error } = useFetch('/api/products');
  if (loading) return <Spinner />;
  if (error) return <Error />;
  return <ProductList products={data} />;
};
```

```tsx
// 2. Redux Toolkit — reduces Redux boilerplate dramatically
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Without RTK: separate action types, action creators, reducers = 100+ lines
// With RTK:
const productsSlice = createSlice({
  name: 'products',
  initialState: { items: [], loading: false },
  reducers: {
    setFilter: (state, action) => { state.filter = action.payload; },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, state => { state.loading = true; })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      });
  },
});
```

---

### How do you handle the CSS-in-JS performance overhead?

**Problem:** Runtime CSS-in-JS libraries (Styled Components, Emotion) generate styles at runtime, causing:
- Style recalculation overhead
- Blocking server-side rendering
- Larger JS bundles
- FOUC issues

**Solutions:**

```bash
# Zero-runtime alternatives
npm install @vanilla-extract/css          # TypeScript-first, static CSS
npm install linaria                       # CSS extracted at build time
```

```tsx
// vanilla-extract — zero runtime, TypeScript types
import { style } from '@vanilla-extract/css';

export const container = style({
  display: 'flex',
  padding: '16px',
  backgroundColor: 'var(--color-surface)',
});
```

```tsx
// CSS Modules — traditional, zero runtime
import styles from './Card.module.css';

const Card = ({ children }) => (
  <div className={styles.container}>{children}</div>
);
```

```tsx
// Tailwind CSS — utility classes, zero runtime
const Card = ({ children }) => (
  <div className="flex p-4 bg-white rounded-lg shadow">{children}</div>
);
```

---

### How do you handle Testing Complexity in React?

```tsx
// React Testing Library — test behavior, not implementation
import { render, screen, userEvent } from '@testing-library/react';
import { LoginForm } from './LoginForm';

test('submits form with user credentials', async () => {
  const mockLogin = vi.fn().mockResolvedValue({ user: { id: 1 } });
  render(<LoginForm onLogin={mockLogin} />);
  
  await userEvent.type(screen.getByLabelText(/email/i), 'user@example.com');
  await userEvent.type(screen.getByLabelText(/password/i), 'password123');
  await userEvent.click(screen.getByRole('button', { name: /sign in/i }));
  
  expect(mockLogin).toHaveBeenCalledWith({
    email: 'user@example.com',
    password: 'password123',
  });
});

test('shows error message on failed login', async () => {
  const mockLogin = vi.fn().mockRejectedValue(new Error('Invalid credentials'));
  render(<LoginForm onLogin={mockLogin} />);
  
  await userEvent.click(screen.getByRole('button', { name: /sign in/i }));
  
  expect(await screen.findByText(/invalid credentials/i)).toBeInTheDocument();
});
```

---

### How do you fix Hydration Mismatches in SSR with React?

**Problem:** Server-rendered HTML doesn't match what React renders on the client.

**Common causes:**
- Date/time (server and client time differ)
- Math.random() or uuid() calls
- Browser-only APIs (window, localStorage)
- User-agent-based logic

```tsx
// 1. Guard browser-only code with useEffect
const Component = () => {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted) return <div className="skeleton" />;
  return <div>{window.innerWidth}px wide</div>;
};
```

```tsx
// 2. Next.js dynamic import with ssr:false
import dynamic from 'next/dynamic';

const BrowserOnlyChart = dynamic(() => import('./Chart'), { ssr: false });
```

```tsx
// 3. suppressHydrationWarning for known differences
const TimeDisplay = () => (
  <time suppressHydrationWarning>
    {new Date().toLocaleTimeString()}
  </time>
);
```

---

### How do you handle Accessibility gaps in React?

```tsx
// Common fixes:

// 1. Semantic HTML + ARIA
// BAD
<div onClick={handleClick} className="button">Submit</div>

// GOOD
<button type="submit" onClick={handleClick}>Submit</button>

// 2. Focus management in modals
const Modal = ({ isOpen, onClose, children }) => {
  const closeButtonRef = useRef(null);
  
  useEffect(() => {
    if (isOpen) closeButtonRef.current?.focus();
  }, [isOpen]);
  
  if (!isOpen) return null;
  return (
    <div role="dialog" aria-modal="true">
      {children}
      <button ref={closeButtonRef} onClick={onClose}>Close</button>
    </div>
  );
};

// 3. Skip links for keyboard navigation
<a href="#main-content" className="skip-link">Skip to main content</a>
<main id="main-content">...</main>
```

```bash
# Automated a11y testing
npm install --save-dev jest-axe eslint-plugin-jsx-a11y
```

---

### How do you handle Form complexity in React?

```tsx
// React Hook Form — performant, minimal re-renders
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
});

const LoginForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({ resolver: zodResolver(schema) });
  
  const onSubmit = async (data) => {
    await loginUser(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        type="email"
        {...register('email')}
        aria-invalid={!!errors.email}
        aria-describedby="email-error"
      />
      {errors.email && <span id="email-error" role="alert">{errors.email.message}</span>}
      
      <input
        type="password"
        {...register('password')}
        aria-invalid={!!errors.password}
      />
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in...' : 'Sign In'}
      </button>
    </form>
  );
};
```

---

### How do you handle Animation performance in React?

```tsx
// Framer Motion — GPU-accelerated animations
import { motion, AnimatePresence } from 'framer-motion';

const Modal = ({ isOpen, children }) => (
  <AnimatePresence>
    {isOpen && (
      <motion.div
        initial={{ opacity: 0, y: -20 }}
        animate={{ opacity: 1, y: 0 }}
        exit={{ opacity: 0, y: -20 }}
        transition={{ type: 'spring', stiffness: 300, damping: 30 }}
      >
        {children}
      </motion.div>
    )}
  </AnimatePresence>
);
```

```tsx
// CSS transitions — most performant, hardware-accelerated
// Only animate: transform, opacity, filter (compositor-only properties)
const Card = ({ expanded }) => (
  <div
    style={{
      transform: `scaleY(${expanded ? 1 : 0})`,
      opacity: expanded ? 1 : 0,
      transition: 'transform 200ms ease, opacity 200ms ease',
    }}
  />
);
```

---

### How do you use Concurrent Mode features to improve UX?

```tsx
// useTransition — mark state updates as non-urgent
const SearchResults = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const handleSearch = (e) => {
    setQuery(e.target.value); // urgent: update input immediately
    
    startTransition(() => {
      // non-urgent: can be deferred
      const filtered = heavyFilter(allData, e.target.value);
      setResults(filtered);
    });
  };
  
  return (
    <>
      <input value={query} onChange={handleSearch} />
      {isPending && <span>Updating...</span>}
      <ResultList results={results} />
    </>
  );
};
```

```tsx
// useDeferredValue — defer expensive renders
const ExpensiveList = React.memo(({ query }) => {
  // expensive filtering
  const filtered = allItems.filter(item => item.name.includes(query));
  return <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
});

const App = () => {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ExpensiveList query={deferredQuery} />
    </>
  );
};
```

---

### How do you implement Error Handling in React?

```tsx
// Error Boundary — catch JS errors in component tree
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack);
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>Something went wrong.</div>;
    }
    return this.props.children;
  }
}

// With react-error-boundary library (hooks-based)
import { ErrorBoundary } from 'react-error-boundary';

const App = () => (
  <ErrorBoundary
    FallbackComponent={({ error, resetErrorBoundary }) => (
      <div>
        <p>Error: {error.message}</p>
        <button onClick={resetErrorBoundary}>Try again</button>
      </div>
    )}
    onError={(error, info) => logToService(error, info)}
  >
    <FeatureComponent />
  </ErrorBoundary>
);
```

---

### How do you handle TypeScript integration challenges in React?

```tsx
// Generic components
interface SelectProps<T> {
  items: T[];
  value: T;
  onChange: (value: T) => void;
  getLabel: (item: T) => string;
  getValue: (item: T) => string | number;
}

function Select<T>({ items, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select value={String(getValue(value))} onChange={e => {
      const item = items.find(i => String(getValue(i)) === e.target.value);
      if (item) onChange(item);
    }}>
      {items.map(item => (
        <option key={String(getValue(item))} value={String(getValue(item))}>
          {getLabel(item)}
        </option>
      ))}
    </select>
  );
}

// Utility types for events
type InputChangeEvent = React.ChangeEvent<HTMLInputElement>;
type ButtonClickEvent = React.MouseEvent<HTMLButtonElement>;

// forwardRef with TypeScript
const Input = React.forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement>>(
  ({ className, ...props }, ref) => (
    <input ref={ref} className={`input ${className}`} {...props} />
  )
);
```

---

### How do you separate Global State from Server State?

**The confusion:** Putting server data (API responses) into global state (Redux) leads to stale data, synchronization issues, and over-complicated reducers.

**Solution: TanStack Query for server state:**

```tsx
// Server state → TanStack Query (handles caching, revalidation, loading)
const useProducts = (filter) => useQuery({
  queryKey: ['products', filter],
  queryFn: () => fetchProducts(filter),
  staleTime: 5 * 60 * 1000, // 5 minutes
});

// UI state → Zustand or local state
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set(state => ({ sidebarOpen: !state.sidebarOpen })),
}));

const Products = () => {
  // Server state
  const { data, isLoading, error } = useProducts('electronics');
  
  // UI state
  const { sidebarOpen, toggleSidebar } = useUIStore();
  
  return (/* ... */);
};
```

---

### How do you handle React dependency management and upgrade pain?

```bash
# Use codemods for automated upgrades
npx codemod react/19/migration-guide

# Check peer dependency compatibility
npm info react-router peerDependencies

# Gradual upgrade with strict peer deps flag
npm install react@19 --legacy-peer-deps

# Use Renovate or Dependabot for automated dependency PRs
```

```json
// .github/dependabot.yml
{
  "version": 2,
  "updates": [
    {
      "package-ecosystem": "npm",
      "directory": "/",
      "schedule": { "interval": "weekly" },
      "groups": {
        "react-ecosystem": {
          "patterns": ["react", "react-dom", "@types/react*"]
        }
      }
    }
  ]
}
```

---

## React Anti-Patterns & How to Fix Them

### What is the "Mutating State Directly" anti-pattern?

```tsx
// BAD — mutates state directly
const addItem = (item) => {
  cart.push(item); // mutation! React won't detect this change
  setCart(cart);   // same reference, no re-render
};

// GOOD — create new reference
const addItem = (item) => {
  setCart(prev => [...prev, item]);
};

// BAD — mutating object state
const updateUser = (name) => {
  user.name = name; // mutation!
  setUser(user);
};

// GOOD
const updateUser = (name) => {
  setUser(prev => ({ ...prev, name }));
};
```

---

### Why shouldn't you use array index as key?

```tsx
// BAD — using index as key causes state mismatches on reorder/delete
const List = ({ items }) => (
  <ul>
    {items.map((item, index) => (
      <li key={index}><input defaultValue={item.name} /></li> // input state gets confused
    ))}
  </ul>
);

// GOOD — use stable, unique IDs
const List = ({ items }) => (
  <ul>
    {items.map(item => (
      <li key={item.id}><input defaultValue={item.name} /></li>
    ))}
  </ul>
);
```

**Why it's a problem:** When items are reordered or deleted, React uses keys to match old and new elements. Index-based keys lead to incorrect reconciliation, especially with stateful child components (inputs, animations).

---

### What is "Overusing useEffect"?

```tsx
// BAD — useEffect for derived state (unnecessary)
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]); // extra render!

// GOOD — compute derived value during render
const fullName = `${firstName} ${lastName}`;

// BAD — useEffect for event-driven logic
useEffect(() => {
  if (submitted) {
    sendAnalytics('form-submitted');
  }
}, [submitted]);

// GOOD — handle in the event handler
const handleSubmit = () => {
  setSubmitted(true);
  sendAnalytics('form-submitted'); // right place for it
};
```

---

### What is a "God Component" and how do you fix it?

```tsx
// BAD — God Component: does everything
const ProductPage = () => {
  const [products, setProducts] = useState([]);
  const [filter, setFilter] = useState('all');
  const [sort, setSort] = useState('price');
  const [cart, setCart] = useState([]);
  const [user, setUser] = useState(null);
  const [searchQuery, setSearchQuery] = useState('');
  // 200 more lines...
};

// GOOD — split into smaller, focused components
const ProductPage = () => (
  <ProductPageLayout>
    <ProductSearch />
    <ProductFilters />
    <ProductGrid />
    <CartSummary />
  </ProductPageLayout>
);

// Each subcomponent manages its own state
const ProductSearch = () => {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
};
```

---

### What is the "Not Using Composition Properly" anti-pattern?

```tsx
// BAD — passing many props to configure behavior
<Card
  hasHeader={true}
  headerTitle="Products"
  hasFooter={true}
  footerContent={<Button>View All</Button>}
  hasImage={true}
  imageUrl="/img.jpg"
/>

// GOOD — composition pattern
<Card>
  <Card.Header>Products</Card.Header>
  <Card.Image src="/img.jpg" />
  <Card.Body>Content here</Card.Body>
  <Card.Footer>
    <Button>View All</Button>
  </Card.Footer>
</Card>
```

---

### What is the "Fetching Data in the Wrong Place" anti-pattern?

```tsx
// BAD — fetching in render (race conditions, no caching, prop drilling)
const ProductList = ({ categoryId }) => {
  const [products, setProducts] = useState([]);
  
  useEffect(() => {
    fetch(`/api/products?category=${categoryId}`)
      .then(r => r.json())
      .then(setProducts);
  }, [categoryId]);
  
  return <div>{products.map(p => <Card key={p.id} {...p} />)}</div>;
};

// GOOD — use TanStack Query
const ProductList = ({ categoryId }) => {
  const { data: products, isLoading } = useQuery({
    queryKey: ['products', categoryId],
    queryFn: () => fetchProducts(categoryId),
  });
  
  if (isLoading) return <Skeleton />;
  return <div>{products?.map(p => <Card key={p.id} {...p} />)}</div>;
};
```

---

### What is the "Overusing Context for frequently changing values" anti-pattern?

```tsx
// BAD — Context re-renders ALL consumers on every change
const MouseContext = createContext({ x: 0, y: 0 });

const MouseProvider = ({ children }) => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  // Every mouse move → re-renders every context consumer!
  useEffect(() => {
    const handler = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  
  return <MouseContext.Provider value={position}>{children}</MouseContext.Provider>;
};

// GOOD — use Zustand with selector for granular subscriptions
const useMouseStore = create((set) => ({
  x: 0,
  y: 0,
  setPosition: (x, y) => set({ x, y }),
}));

// Component only re-renders when x changes, not y
const XDisplay = () => {
  const x = useMouseStore(state => state.x);
  return <span>X: {x}</span>;
};
```

---

## React vs Alternatives

### How does React compare to Vue, Angular, and Svelte?

| Feature | React | Vue | Angular | Svelte |
|---|---|---|---|---|
| **Type** | UI library | Progressive framework | Full framework | Compiler |
| **Learning curve** | Medium | Easy | Steep | Easy |
| **Bundle size** | ~45KB (gzip) | ~34KB | ~140KB | ~1.6KB runtime |
| **Performance** | Good (VDOM) | Good (VDOM) | Good | Excellent (no VDOM) |
| **State management** | External (Redux, Zustand) | Built-in (Pinia) | Built-in (RxJS) | Built-in (stores) |
| **TypeScript** | Excellent | Good | Excellent (first-class) | Good |
| **SSR** | Next.js | Nuxt | Angular Universal | SvelteKit |
| **Ecosystem** | Largest | Large | Large | Growing |
| **Corporate backing** | Meta | Community + Evan You | Google | Rich Harris (Vercel) |
| **Job market** | Largest | Large | Large | Small |
| **Data binding** | One-way | Two-way (v-model) | Two-way (ngModel) | Two-way |
| **Best for** | Large apps, flexibility | Mid-size apps | Enterprise | Performance-critical |

---

### When should you choose React vs alternatives?

**Choose React when:**
- Large team with varied frontend experience
- Rich ecosystem is a priority
- Building complex, data-heavy applications
- Integration with existing React codebases
- Strong TypeScript needs

**Choose Vue when:**
- Team is newer to frontend frameworks
- Quick setup for mid-size applications
- Progressive adoption in a non-SPA app (e.g., adding to a PHP project)

**Choose Angular when:**
- Large enterprise team with strict conventions
- Full-featured framework needed (routing, HTTP, forms, DI built-in)
- Long-term maintenance is a priority
- Team knows TypeScript well

**Choose Svelte/SvelteKit when:**
- Performance is critical (minimal runtime)
- Building smaller, content-focused sites
- Team wants simpler reactivity model
- Bundle size is a top constraint

---

[← Back to README](./README.md)
