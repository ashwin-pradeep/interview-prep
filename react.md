# React Fundamentals & Advanced

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Component Patterns](#component-patterns)
3. [Hooks Deep Dive](#hooks-deep-dive)
4. [State & Lifecycle](#state--lifecycle)
5. [Performance Optimization](#performance-optimization)
6. [Routing & Navigation](#routing--navigation)
7. [Forms & Controlled Components](#forms--controlled-components)
8. [Context & Prop Drilling](#context--prop-drilling)
9. [Error Handling](#error-handling)
10. [Server-Side Rendering & Frameworks](#server-side-rendering--frameworks)
11. [Testing React Components](#testing-react-components)
12. [Advanced Patterns & Architecture](#advanced-patterns--architecture)

---

## Core Concepts

### What is React and why use it?

React is a **declarative, component-based** JavaScript library for building user interfaces. Created by Facebook (Meta) in 2013.

**Key reasons to use React:**

| Feature | Benefit |
|---|---|
| Virtual DOM | Efficient updates — diffs against real DOM |
| Component-based | Reusable, composable UI building blocks |
| Unidirectional data flow | Predictable state management |
| Rich ecosystem | React Router, Redux, Next.js, etc. |
| Large community | Extensive libraries, tools, hiring pool |

```jsx
// Declarative — describe what the UI should look like
function Welcome({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Imperative equivalent (vanilla JS)
function renderWelcome(name) {
  const h1 = document.createElement("h1");
  h1.textContent = `Hello, ${name}!`;
  document.body.appendChild(h1);
}
```

---

### What is JSX?

JSX (JavaScript XML) is a syntax extension that lets you write HTML-like markup inside JavaScript. It compiles to `React.createElement()` calls.

```jsx
// JSX
const element = <h1 className="title">Hello</h1>;

// Compiles to
const element = React.createElement("h1", { className: "title" }, "Hello");
```

**JSX rules:**
- Must return a single root element (use `<>...</>` fragments)
- Use `className` instead of `class`
- Use `htmlFor` instead of `for`
- All tags must be closed (`<img />`, `<br />`)
- JavaScript expressions go inside `{}`
- Comments: `{/* comment */}`

```jsx
function App() {
  const items = ["React", "Vue", "Angular"];
  return (
    <>
      <h1>Frameworks</h1>
      <ul>
        {items.map((item, i) => (
          <li key={i}>{item}</li>
        ))}
      </ul>
    </>
  );
}
```

---

### What is the Virtual DOM and how does it work?

The **Virtual DOM** is a lightweight in-memory representation of the real DOM. React uses it to batch and optimize updates.

```
State Change → New Virtual DOM → Diff (Reconciliation) → Minimal Real DOM Updates
```

**Reconciliation process:**
1. State changes trigger a re-render
2. React creates a new Virtual DOM tree
3. React **diffs** the new tree against the previous one
4. Only the **changed nodes** are updated in the real DOM (commit phase)

```jsx
// When count changes, React:
// 1. Re-runs Counter() to get new VDOM
// 2. Diffs <span>0</span> vs <span>1</span>
// 3. Only updates the text node in real DOM
function Counter() {
  const [count, setCount] = useState(0);
  return <span>{count}</span>;
}
```

**Key performance aspect:** Batching multiple state updates into a single re-render (automatic in React 18+).

---

### What is reconciliation in React?

Reconciliation is React's **diffing algorithm** that determines the minimum number of DOM operations to update the UI.

**Heuristics used:**
1. **Different element types** → tear down old tree, build new one
2. **Same element type** → update attributes, recurse on children
3. **Keys** → match children efficiently in lists

```jsx
// Different types — full remount
<div><Counter /></div>   →   <span><Counter /></span>
// Counter state is lost, component remounts

// Same type — attribute update only
<div className="a" />    →   <div className="b" />
// Only className changes in DOM

// Keys help with list reordering
<ul>
  <li key="a">A</li>      →   <li key="c">C</li>
  <li key="b">B</li>           <li key="a">A</li>
  <li key="c">C</li>           <li key="b">B</li>
</ul>
// React knows to move, not recreate
```

---

### What is the difference between React elements and components?

| Concept | Description | Example |
|---|---|---|
| **Element** | Plain object describing what to render | `<div />` or `React.createElement("div")` |
| **Component** | Function or class that returns elements | `function App() { return <div /> }` |

```jsx
// Element — immutable description
const element = <h1>Hello</h1>;
// { type: 'h1', props: { children: 'Hello' } }

// Component — reusable function that produces elements
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

// Usage — React calls Greeting() to get the element
<Greeting name="Alice" />
```

---

### What are React Fragments?

Fragments let you group children without adding extra DOM nodes.

```jsx
// Problem — unnecessary wrapper div
function Table() {
  return (
    <div>  {/* Extra DOM node! */}
      <td>A</td>
      <td>B</td>
    </div>
  );
}

// Solution — Fragment
function Table() {
  return (
    <>
      <td>A</td>
      <td>B</td>
    </>
  );
}

// Long syntax (needed for key prop)
function List({ items }) {
  return items.map((item) => (
    <React.Fragment key={item.id}>
      <dt>{item.term}</dt>
      <dd>{item.desc}</dd>
    </React.Fragment>
  ));
}
```

---

### What is the difference between controlled and uncontrolled components?

| Feature | Controlled | Uncontrolled |
|---|---|---|
| Data source | React state | DOM (ref) |
| Updates | `onChange` handler | Direct DOM access |
| Validation | On every keystroke | On submit |
| Use case | Most forms | File inputs, quick prototypes |

```jsx
// Controlled
function ControlledInput() {
  const [value, setValue] = useState("");
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

// Uncontrolled
function UncontrolledInput() {
  const inputRef = useRef(null);
  const handleSubmit = () => {
    console.log(inputRef.current.value);
  };
  return <input ref={inputRef} defaultValue="" />;
}
```

---

### What are keys in React and why are they important?

Keys help React identify which items in a list have changed, been added, or removed. They must be **stable, unique, and predictable**.

```jsx
// ✅ Good — stable unique ID
{users.map((user) => (
  <UserCard key={user.id} user={user} />
))}

// ❌ Bad — index as key (breaks with reordering/deletion)
{users.map((user, index) => (
  <UserCard key={index} user={user} />
))}

// ❌ Bad — random key (remounts every render)
{users.map((user) => (
  <UserCard key={Math.random()} user={user} />
))}
```

**When index is okay:** Static lists that never reorder, filter, or change.

---

## Component Patterns

### What is the difference between functional and class components?

| Feature | Functional | Class |
|---|---|---|
| Syntax | Plain function | `class extends React.Component` |
| State | `useState` hook | `this.state` |
| Lifecycle | `useEffect` hook | `componentDidMount`, etc. |
| `this` binding | Not needed | Required for methods |
| Performance | Slightly lighter | Slightly heavier |
| Current status | **Recommended** | Legacy (still supported) |

```jsx
// Functional (modern)
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Class (legacy)
class Counter extends React.Component {
  state = { count: 0 };
  render() {
    return (
      <button onClick={() => this.setState({ count: this.state.count + 1 })}>
        {this.state.count}
      </button>
    );
  }
}
```

---

### What are Higher-Order Components (HOCs)?

A HOC is a function that takes a component and returns a new component with enhanced behavior.

```jsx
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { isLoggedIn } = useAuth();

    if (!isLoggedIn) return <Redirect to="/login" />;
    return <WrappedComponent {...props} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);
```

**Conventions:**
- Name: `withXxx`
- Pass through all props
- Don't mutate the original component

**Drawbacks:** Wrapper hell, naming collisions, hard to trace props. Modern alternative: **custom hooks**.

---

### What is the Render Props pattern?

A component with a render prop takes a function that returns a React element, allowing dynamic rendering behavior.

```jsx
function MouseTracker({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e) => {
    setPos({ x: e.clientX, y: e.clientY });
  };

  return <div onMouseMove={handleMouseMove}>{render(pos)}</div>;
}

// Usage
<MouseTracker render={({ x, y }) => <p>Mouse: {x}, {y}</p>} />
```

**Modern alternative:** Custom hooks achieve the same code reuse more cleanly.

```jsx
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);
  return pos;
}
```

---

### What are Compound Components?

Compound components work together to form a complete UI while sharing implicit state.

```jsx
function Tabs({ children, defaultIndex = 0 }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);
  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.TabList = function TabList({ children }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Tab = function Tab({ index, children }) {
  const { activeIndex, setActiveIndex } = useContext(TabsContext);
  return (
    <button
      role="tab"
      aria-selected={activeIndex === index}
      onClick={() => setActiveIndex(index)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function Panel({ index, children }) {
  const { activeIndex } = useContext(TabsContext);
  return activeIndex === index ? <div role="tabpanel">{children}</div> : null;
};

// Usage — flexible, readable API
<Tabs>
  <Tabs.TabList>
    <Tabs.Tab index={0}>Tab 1</Tabs.Tab>
    <Tabs.Tab index={1}>Tab 2</Tabs.Tab>
  </Tabs.TabList>
  <Tabs.Panel index={0}>Content 1</Tabs.Panel>
  <Tabs.Panel index={1}>Content 2</Tabs.Panel>
</Tabs>
```

---

### What is the Container/Presentational pattern?

| Role | Container | Presentational |
|---|---|---|
| Concern | Logic, data fetching, state | UI rendering |
| State | Stateful | Stateless (usually) |
| Hooks | Yes | Props only |
| Reusability | Low | High |

```jsx
// Container — handles logic
function UserListContainer() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    fetch("/api/users").then((r) => r.json()).then(setUsers);
  }, []);
  return <UserList users={users} />;
}

// Presentational — handles display
function UserList({ users }) {
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

**Modern take:** Custom hooks replace containers; presentational components remain useful for design systems.

---

## Hooks Deep Dive

### What are React Hooks and what rules do they follow?

Hooks let you use state and lifecycle features in functional components (introduced in React 16.8).

**Rules of Hooks:**
1. **Only call hooks at the top level** — not inside loops, conditions, or nested functions
2. **Only call hooks from React functions** — components or custom hooks

```jsx
// ✅ Correct
function Component() {
  const [count, setCount] = useState(0);
  useEffect(() => { /* ... */ }, []);
  return <div>{count}</div>;
}

// ❌ Wrong — conditional hook
function Component({ isLogged }) {
  if (isLogged) {
    const [user, setUser] = useState(null); // Breaks rules!
  }
}
```

**Why?** React relies on hook call order to map state to the correct hook. Conditional hooks break this mapping.

---

### Explain useState in depth.

`useState` declares a state variable. Returns `[currentValue, setterFunction]`.

```jsx
const [count, setCount] = useState(0);

// Direct update
setCount(5);

// Functional update (when new state depends on old)
setCount((prev) => prev + 1);

// Lazy initialization (expensive computation)
const [data, setData] = useState(() => {
  return JSON.parse(localStorage.getItem("data"));
});
```

**Key behaviors:**
- Setter triggers a re-render
- State updates are **batched** in React 18+ (even in timeouts/promises)
- State is **replaced**, not merged (unlike `this.setState`)
- Object state requires spreading: `setState(prev => ({ ...prev, name: "new" }))`

---

### Explain useEffect in depth.

`useEffect` handles side effects: data fetching, subscriptions, DOM mutations.

```jsx
// Runs after every render
useEffect(() => {
  console.log("rendered");
});

// Runs once on mount
useEffect(() => {
  fetchData();
}, []);

// Runs when dependency changes
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// Cleanup (runs before next effect and on unmount)
useEffect(() => {
  const sub = eventBus.subscribe(handler);
  return () => sub.unsubscribe(); // cleanup
}, [handler]);
```

**Common pitfalls:**
- Missing dependencies → stale closures
- Object/array dependencies → infinite loops (use primitive values or `useMemo`)
- Fetching without cleanup → race conditions

```jsx
// Avoiding race conditions
useEffect(() => {
  let cancelled = false;
  fetch(`/api/user/${id}`)
    .then((r) => r.json())
    .then((data) => {
      if (!cancelled) setUser(data);
    });
  return () => { cancelled = true; };
}, [id]);
```

---

### Explain useRef and its use cases.

`useRef` returns a mutable ref object whose `.current` property persists across renders without causing re-renders.

```jsx
// DOM access
function TextInput() {
  const inputRef = useRef(null);
  const focus = () => inputRef.current.focus();
  return <input ref={inputRef} />;
}

// Mutable value that doesn't trigger re-render
function Timer() {
  const intervalRef = useRef(null);
  const start = () => {
    intervalRef.current = setInterval(() => console.log("tick"), 1000);
  };
  const stop = () => clearInterval(intervalRef.current);
  return <button onClick={stop}>Stop</button>;
}

// Previous value tracking
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

---

### What is useCallback and when should you use it?

`useCallback` memoizes a function reference to prevent unnecessary re-renders of child components.

```jsx
// Without useCallback — new function every render
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount((c) => c + 1); // new ref each render
  return <Child onClick={handleClick} />;
}

// With useCallback — stable function reference
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => {
    setCount((c) => c + 1);
  }, []);
  return <Child onClick={handleClick} />;
}

const Child = React.memo(({ onClick }) => {
  console.log("Child render");
  return <button onClick={onClick}>Click</button>;
});
```

**When to use:** Only when passing callbacks to `React.memo`-wrapped children or as dependencies of other hooks. Don't wrap every function.

---

### What is useMemo and when should you use it?

`useMemo` memoizes the **result** of an expensive computation.

```jsx
function FilteredList({ items, query }) {
  // Recomputes only when items or query change
  const filtered = useMemo(() => {
    return items.filter((item) =>
      item.name.toLowerCase().includes(query.toLowerCase())
    );
  }, [items, query]);

  return <List data={filtered} />;
}
```

**useMemo vs useCallback:**

| Hook | Memoizes | Returns |
|---|---|---|
| `useMemo` | Computed value | The value |
| `useCallback` | Function definition | The function |

```jsx
useMemo(() => fn, [deps])    // equivalent to:
useCallback(fn, [deps])      // (for function memoization)
```

**Don't overuse:** React's re-rendering is fast. Only memoize expensive computations or reference-sensitive dependencies.

---

### What is useReducer and when to prefer it over useState?

`useReducer` manages complex state logic with a reducer function, similar to Redux.

```jsx
const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <div>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

**Prefer useReducer when:**
- State has multiple sub-values
- Next state depends on previous state
- Actions are complex and need to be testable
- State logic should be extracted outside the component

---

### What is useLayoutEffect?

`useLayoutEffect` fires **synchronously** after DOM mutations but **before** the browser paints. Useful for measuring DOM elements.

```jsx
function Tooltip({ targetRef, children }) {
  const [pos, setPos] = useState({ top: 0, left: 0 });
  const tooltipRef = useRef(null);

  // Runs before paint — no visual flicker
  useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    setPos({ top: rect.bottom + 8, left: rect.left });
  }, [targetRef]);

  return (
    <div ref={tooltipRef} style={{ position: "fixed", ...pos }}>
      {children}
    </div>
  );
}
```

| Hook | Timing | Use Case |
|---|---|---|
| `useEffect` | After paint (async) | Data fetching, subscriptions |
| `useLayoutEffect` | Before paint (sync) | DOM measurements, preventing flicker |

---

### What are custom hooks?

Custom hooks are functions starting with `use` that encapsulate reusable stateful logic.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(url)
      .then((res) => res.json())
      .then((json) => {
        if (!cancelled) { setData(json); setLoading(false); }
      })
      .catch((err) => {
        if (!cancelled) { setError(err); setLoading(false); }
      });
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <Profile user={user} />;
}
```

---

### What are useId, useDeferredValue, and useTransition (React 18)?

```jsx
// useId — generates unique IDs for accessibility
function FormField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" />
    </>
  );
}

// useTransition — marks state updates as non-urgent
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value); // urgent — update input immediately
    startTransition(() => {
      setResults(filterLargeList(e.target.value)); // non-urgent
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
    </>
  );
}

// useDeferredValue — defers re-rendering with a stale value
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => filterLargeList(deferredQuery), [deferredQuery]);
  return <ResultsList results={results} />;
}
```

---

## State & Lifecycle

### What is the React component lifecycle?

```
Mounting           Updating              Unmounting
  │                  │                      │
  ▼                  ▼                      ▼
constructor()     getDerivedState()      componentWillUnmount()
  │              shouldComponentUpdate()
render()           render()
  │                  │
componentDidMount() componentDidUpdate()
```

**Hooks equivalents:**

| Lifecycle Method | Hook Equivalent |
|---|---|
| `componentDidMount` | `useEffect(() => {}, [])` |
| `componentDidUpdate` | `useEffect(() => {}, [dep])` |
| `componentWillUnmount` | `useEffect(() => { return cleanup }, [])` |
| `shouldComponentUpdate` | `React.memo` |
| `getDerivedStateFromProps` | `useState` + early return |

---

### How does state batching work in React 18?

React 18 introduces **automatic batching** for all updates, not just event handlers.

```jsx
// React 17 — only batched inside event handlers
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // ✅ Single re-render (batched)
}

setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // ❌ Two re-renders in React 17!
}, 1000);

// React 18 — batched everywhere
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // ✅ Single re-render (automatic batching)
}, 1000);

// Opt out of batching (rare)
import { flushSync } from "react-dom";
flushSync(() => setCount(c => c + 1)); // renders immediately
flushSync(() => setFlag(f => !f));      // renders again
```

---

### What is lifting state up?

Moving shared state to the nearest common ancestor so sibling components can share it.

```jsx
function Parent() {
  const [temperature, setTemperature] = useState("");

  return (
    <div>
      <CelsiusInput temp={temperature} onTempChange={setTemperature} />
      <FahrenheitInput temp={temperature} onTempChange={setTemperature} />
      <BoilingVerdict celsius={parseFloat(temperature)} />
    </div>
  );
}

function CelsiusInput({ temp, onTempChange }) {
  return (
    <input
      value={temp}
      onChange={(e) => onTempChange(e.target.value)}
      placeholder="Celsius"
    />
  );
}
```

**When to lift:** When two or more components need the same changing data.

---

## Performance Optimization

### What is React.memo?

`React.memo` is a higher-order component that memoizes the rendered output. It skips re-rendering if props haven't changed (shallow comparison).

```jsx
const ExpensiveList = React.memo(function ExpensiveList({ items }) {
  console.log("Rendering list...");
  return items.map((item) => <div key={item.id}>{item.name}</div>);
});

// Custom comparison
const MemoizedComponent = React.memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id; // return true to skip re-render
});
```

**When to use:**
- Component renders often with same props
- Component is expensive to render
- Parent re-renders frequently

**When NOT to use:**
- Component always receives new props
- Component is cheap to render

---

### What is React.lazy and Suspense?

Code splitting with lazy loading reduces initial bundle size.

```jsx
const LazyDashboard = React.lazy(() => import("./Dashboard"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <LazyDashboard />
    </Suspense>
  );
}

// Route-based splitting
function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<LazyDashboard />} />
        <Route path="/settings" element={<LazySettings />} />
      </Routes>
    </Suspense>
  );
}
```

**Best practices:**
- Split at route level first
- Use named exports with `React.lazy(() => import("./X").then(m => ({ default: m.NamedExport })))`
- Preload on hover: `const preload = () => import("./Dashboard")`

---

### How do you prevent unnecessary re-renders?

| Technique | What it does |
|---|---|
| `React.memo` | Skip re-render if props unchanged |
| `useMemo` | Memoize expensive computations |
| `useCallback` | Memoize function references |
| Key stability | Avoid random/index keys |
| State colocation | Keep state close to where it's used |
| Context splitting | Separate contexts for different data |
| Virtualization | Only render visible list items |

```jsx
// State colocation — move state down
// ❌ Entire App re-renders on hover
function App() {
  const [hovered, setHovered] = useState(false);
  return (
    <>
      <HoverBox hovered={hovered} onHover={setHovered} />
      <ExpensiveSidebar />
    </>
  );
}

// ✅ Only HoverBox re-renders
function App() {
  return (
    <>
      <HoverBox />
      <ExpensiveSidebar />
    </>
  );
}
function HoverBox() {
  const [hovered, setHovered] = useState(false);
  return <div onMouseEnter={() => setHovered(true)}>{/* ... */}</div>;
}
```

---

### What is windowing / virtualization?

Rendering only visible items in long lists instead of the entire dataset.

```jsx
import { FixedSizeList } from "react-window";

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Libraries:** `react-window` (lightweight), `react-virtuoso` (feature-rich), `@tanstack/virtual` (headless).

---

## Routing & Navigation

### How does React Router work?

React Router provides declarative, component-based routing for React apps.

```jsx
import { BrowserRouter, Routes, Route, Link, useParams, useNavigate } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/users">Users</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users" element={<UserList />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}

function UserDetail() {
  const { id } = useParams();
  const navigate = useNavigate();
  return (
    <div>
      <h1>User {id}</h1>
      <button onClick={() => navigate("/users")}>Back</button>
    </div>
  );
}
```

---

### What are protected/private routes?

```jsx
function PrivateRoute({ children }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}

// Usage
<Route
  path="/dashboard"
  element={
    <PrivateRoute>
      <Dashboard />
    </PrivateRoute>
  }
/>
```

---

## Forms & Controlled Components

### How do you handle forms in React?

```jsx
function ContactForm() {
  const [form, setForm] = useState({ name: "", email: "", message: "" });
  const [errors, setErrors] = useState({});

  const validate = () => {
    const newErrors = {};
    if (!form.name) newErrors.name = "Name is required";
    if (!form.email.includes("@")) newErrors.email = "Invalid email";
    return newErrors;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const newErrors = validate();
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    submitForm(form);
  };

  const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={form.name} onChange={handleChange} />
      {errors.name && <span className="error">{errors.name}</span>}
      <input name="email" value={form.email} onChange={handleChange} />
      {errors.email && <span className="error">{errors.email}</span>}
      <textarea name="message" value={form.message} onChange={handleChange} />
      <button type="submit">Send</button>
    </form>
  );
}
```

**Form libraries:** React Hook Form (performance-focused), Formik (feature-rich), Zod/Yup (validation schemas).

---

### What is React Hook Form and why use it?

React Hook Form minimizes re-renders by using uncontrolled inputs with refs.

```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "Min 8 characters"),
});

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}
      <input type="password" {...register("password")} />
      {errors.password && <span>{errors.password.message}</span>}
      <button type="submit">Login</button>
    </form>
  );
}
```

---

## Context & Prop Drilling

### What is prop drilling and how do you avoid it?

Prop drilling is passing data through many component layers that don't need it.

```jsx
// ❌ Prop drilling
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <Avatar user={user} />  {/* Only this needs user! */}
    </Sidebar>
  </Layout>
</App>

// ✅ Context API
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={user}>
      <Layout>
        <Sidebar>
          <Avatar />
        </Sidebar>
      </Layout>
    </UserContext.Provider>
  );
}

function Avatar() {
  const user = useContext(UserContext);
  return <img src={user.avatar} alt={user.name} />;
}
```

**Other solutions:** Component composition, render props, state management libraries.

---

## Error Handling

### What are Error Boundaries?

Error boundaries are class components that catch JavaScript errors in their child component tree.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <FallbackUI error={this.state.error} />;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

**Limitations:**
- Only class components (no hook equivalent yet)
- Don't catch: event handlers, async code, SSR, errors in the boundary itself

```jsx
// For event handlers — use try/catch
function Button() {
  const [error, setError] = useState(null);
  const handleClick = () => {
    try {
      riskyOperation();
    } catch (e) {
      setError(e);
    }
  };
  if (error) return <ErrorMessage error={error} />;
  return <button onClick={handleClick}>Click</button>;
}
```

---

## Server-Side Rendering & Frameworks

### What is SSR vs CSR vs SSG vs ISR?

| Strategy | Rendering | When HTML Generated | Use Case |
|---|---|---|---|
| **CSR** | Client | In browser at runtime | SPAs, dashboards |
| **SSR** | Server | On each request | Dynamic pages, SEO-critical |
| **SSG** | Build time | At build time | Blogs, docs, marketing |
| **ISR** | Hybrid | Build + revalidation | E-commerce, frequently updated |

```jsx
// Next.js SSR
export async function getServerSideProps() {
  const data = await fetchData();
  return { props: { data } };
}

// Next.js SSG
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data }, revalidate: 60 }; // ISR: revalidate every 60s
}

// Next.js App Router (React Server Components)
async function Page() {
  const data = await fetchData(); // runs on server
  return <div>{data.title}</div>;
}
```

---

### What are React Server Components (RSC)?

RSCs run **only on the server** — they never ship JavaScript to the client.

```jsx
// Server Component (default in Next.js App Router)
async function ProductPage({ id }) {
  const product = await db.query("SELECT * FROM products WHERE id = ?", [id]);
  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCartButton id={id} /> {/* Client Component */}
    </div>
  );
}

// Client Component — needs "use client" directive
"use client";
function AddToCartButton({ id }) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>Add to Cart</button>;
}
```

**Benefits:** Zero client JS for server components, direct database access, smaller bundles.

---

### What is Next.js and when to use it?

Next.js is a React framework providing SSR, SSG, API routes, file-based routing, and more.

| Feature | Details |
|---|---|
| File-based routing | `app/page.tsx` → `/` |
| Server Components | Default in App Router |
| API Routes | `app/api/route.ts` |
| Image optimization | `next/image` with lazy loading |
| Middleware | Edge functions for auth, redirects |
| ISR | Incremental Static Regeneration |

**When to use:** SEO-critical sites, e-commerce, content sites, full-stack React apps.

---

## Testing React Components

### How do you test React components?

```jsx
import { render, screen, fireEvent } from "@testing-library/react";

// Render test
test("renders greeting", () => {
  render(<Greeting name="Alice" />);
  expect(screen.getByText("Hello, Alice!")).toBeInTheDocument();
});

// Interaction test
test("increments counter", () => {
  render(<Counter />);
  const button = screen.getByRole("button");
  fireEvent.click(button);
  expect(screen.getByText("1")).toBeInTheDocument();
});

// Async test
test("loads user data", async () => {
  render(<UserProfile userId="1" />);
  expect(screen.getByText("Loading...")).toBeInTheDocument();
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

**Testing priorities (Testing Library philosophy):**
1. `getByRole` — accessible queries first
2. `getByLabelText` — form elements
3. `getByText` — non-interactive elements
4. `getByTestId` — last resort

---

### How do you test custom hooks?

```jsx
import { renderHook, act } from "@testing-library/react";

test("useCounter increments", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

---

### How do you mock API calls in tests?

```jsx
import { rest } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  rest.get("/api/users/:id", (req, res, ctx) => {
    return res(ctx.json({ id: req.params.id, name: "Alice" }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("fetches and displays user", async () => {
  render(<UserProfile userId="1" />);
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

---

## Advanced Patterns & Architecture

### What is the Provider pattern?

Wrapping your app with providers that supply data/functionality to the component tree.

```jsx
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <QueryClientProvider client={queryClient}>
          <Router>
            <Layout />
          </Router>
        </QueryClientProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// Compose providers to avoid nesting hell
function ComposeProviders({ providers, children }) {
  return providers.reduceRight(
    (acc, [Provider, props]) => <Provider {...props}>{acc}</Provider>,
    children
  );
}

<ComposeProviders
  providers={[
    [ThemeProvider, {}],
    [AuthProvider, {}],
    [QueryClientProvider, { client: queryClient }],
  ]}
>
  <App />
</ComposeProviders>
```

---

### What is the State Machine pattern in React?

Using finite state machines for predictable UI states.

```jsx
const STATES = {
  idle: { FETCH: "loading" },
  loading: { SUCCESS: "success", ERROR: "error" },
  success: { FETCH: "loading" },
  error: { RETRY: "loading" },
};

function reducer(state, event) {
  return STATES[state]?.[event] || state;
}

function DataFetcher() {
  const [state, send] = useReducer(reducer, "idle");
  const [data, setData] = useState(null);

  const fetchData = async () => {
    send("FETCH");
    try {
      const res = await fetch("/api/data");
      setData(await res.json());
      send("SUCCESS");
    } catch {
      send("ERROR");
    }
  };

  return (
    <div>
      {state === "idle" && <button onClick={fetchData}>Load</button>}
      {state === "loading" && <Spinner />}
      {state === "success" && <DataView data={data} />}
      {state === "error" && <button onClick={fetchData}>Retry</button>}
    </div>
  );
}
```

---

### What are portals in React?

Portals render children into a DOM node outside the parent component hierarchy.

```jsx
import { createPortal } from "react-dom";

function Modal({ children, onClose }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById("modal-root")
  );
}

// Events still bubble through React tree (not DOM tree)
function App() {
  const [show, setShow] = useState(false);
  return (
    <div onClick={() => console.log("App clicked")}>
      <button onClick={() => setShow(true)}>Open Modal</button>
      {show && (
        <Modal onClose={() => setShow(false)}>
          <h2>Modal Content</h2>
          <button onClick={() => setShow(false)}>Close</button>
        </Modal>
      )}
    </div>
  );
}
```

---

### What is forwardRef?

`forwardRef` lets a component expose a DOM node to its parent.

```jsx
const FancyInput = React.forwardRef((props, ref) => (
  <input ref={ref} className="fancy-input" {...props} />
));

function Parent() {
  const inputRef = useRef(null);
  return (
    <>
      <FancyInput ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

// With useImperativeHandle — expose custom API
const FancyInput = React.forwardRef((props, ref) => {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ""; },
  }));
  return <input ref={inputRef} {...props} />;
});
```

---

### What is Concurrent React?

Concurrent React lets React prepare multiple UI versions simultaneously, prioritizing urgent updates.

**Key features:**
- **Transitions** — mark non-urgent updates
- **Suspense** — show fallback while loading
- **Streaming SSR** — send HTML incrementally
- **Selective hydration** — hydrate interactive parts first

```jsx
// Transition — keeps UI responsive during expensive updates
function App() {
  const [tab, setTab] = useState("home");
  const [isPending, startTransition] = useTransition();

  const selectTab = (nextTab) => {
    startTransition(() => {
      setTab(nextTab); // non-urgent
    });
  };

  return (
    <div>
      <TabBar onSelect={selectTab} isPending={isPending} />
      <TabContent tab={tab} />
    </div>
  );
}
```

---

### How do you handle internationalization (i18n) in React?

```jsx
import { useTranslation } from "react-i18next";

function Welcome() {
  const { t, i18n } = useTranslation();

  return (
    <div>
      <h1>{t("welcome.title")}</h1>
      <p>{t("welcome.description", { name: "Alice" })}</p>
      <button onClick={() => i18n.changeLanguage("fr")}>Français</button>
    </div>
  );
}

// Translation file (en.json)
// { "welcome": { "title": "Welcome!", "description": "Hello, {{name}}" } }
```

**Libraries:** `react-i18next`, `react-intl` (FormatJS), `next-intl` (Next.js).

---

### What is the difference between React and React Native?

| Feature | React (Web) | React Native (Mobile) |
|---|---|---|
| Target | Browser DOM | iOS/Android native views |
| Elements | `<div>`, `<span>`, `<input>` | `<View>`, `<Text>`, `<TextInput>` |
| Styling | CSS, CSS-in-JS | StyleSheet (Flexbox by default) |
| Navigation | React Router | React Navigation |
| Animations | CSS transitions, Framer Motion | Animated API, Reanimated |

---

### What are the new React 19 features?

| Feature | Description |
|---|---|
| **Actions** | Async functions for form handling |
| **useActionState** | Hook for action state management |
| **useFormStatus** | Pending state for forms |
| **useOptimistic** | Optimistic UI updates |
| **`use` API** | Read promises and context in render |
| **ref as prop** | No more `forwardRef` needed |
| **Document metadata** | `<title>`, `<meta>` in components |

```jsx
// React 19 Actions
function UpdateName() {
  const [error, submitAction, isPending] = useActionState(
    async (prevState, formData) => {
      const error = await updateName(formData.get("name"));
      if (error) return error;
      redirect("/profile");
    },
    null
  );

  return (
    <form action={submitAction}>
      <input name="name" />
      <button disabled={isPending}>Update</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

---

[← Back to README](./README.md)
