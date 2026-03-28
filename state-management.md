# Redux, Context API & State Management

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [State Management Fundamentals](#state-management-fundamentals)
2. [Context API](#context-api)
3. [Redux Core Concepts](#redux-core-concepts)
4. [Redux Toolkit](#redux-toolkit)
5. [Redux Middleware & Async](#redux-middleware--async)
6. [Zustand, Jotai & Recoil](#zustand-jotai--recoil)
7. [State Architecture Patterns](#state-architecture-patterns)
8. [Performance & Best Practices](#performance--best-practices)

---

## State Management Fundamentals

### What are the different types of state in a React application?

| Type | Description | Example | Tool |
|---|---|---|---|
| **Local/UI state** | Component-specific | Form input, toggle, modal open | `useState`, `useReducer` |
| **Server state** | Data from APIs | User list, product catalog | React Query, SWR |
| **Global/App state** | Shared across components | Auth, theme, language | Context, Redux, Zustand |
| **URL state** | Stored in the URL | Filters, pagination, search | React Router, `useSearchParams` |
| **Form state** | Form input + validation | Login form, checkout | React Hook Form, Formik |

**Key principle:** Use the **simplest tool** that fits. Don't put everything in global state.

```
Local State (useState)
  └── Lift State Up
        └── Context API (moderate sharing)
              └── Redux / Zustand (complex global state)
```

---

### When do you need a state management library?

**You need one when:**
- Multiple components at different tree levels need the same data
- State updates follow complex logic (many actions, computed values)
- You need middleware (logging, persistence, async)
- App has significant server state caching needs

**You DON'T need one when:**
- State is local to a component or small subtree
- `useContext` + `useReducer` handles the case
- Server state can be managed by React Query/SWR

---

### What is the difference between client state and server state?

| Aspect | Client State | Server State |
|---|---|---|
| Ownership | Client owns it | Server is source of truth |
| Persistence | In memory (lost on refresh) | Database/API |
| Examples | Theme, sidebar open, form input | Users, products, orders |
| Staleness | Always fresh | Can be stale (needs sync) |
| Tool | Redux, Zustand, Context | React Query, SWR, RTK Query |

```jsx
// Server state with React Query
function UserList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((r) => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <List items={data} />;
}
```

---

## Context API

### How does the Context API work?

Context provides a way to pass data through the component tree without prop drilling.

```jsx
// 1. Create context
const ThemeContext = createContext("light");

// 2. Provide context
function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

// 3. Consume context
function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button
      className={theme}
      onClick={() => setTheme(theme === "light" ? "dark" : "light")}
    >
      Toggle Theme
    </button>
  );
}
```

---

### What are the limitations of Context API?

| Limitation | Description | Solution |
|---|---|---|
| **Re-render problem** | All consumers re-render when value changes | Split contexts, memoize values |
| **No selector** | Can't subscribe to part of context | Use separate contexts or Zustand |
| **Not optimized for frequent updates** | Every update re-renders all consumers | Use external state lib for frequent changes |
| **Boilerplate** | Provider + context + hook per piece of state | Custom hook wrappers |

```jsx
// ❌ Problem — changing theme re-renders UserInfo consumers too
const AppContext = createContext();
<AppContext.Provider value={{ theme, user, notifications }}>

// ✅ Solution — split contexts
const ThemeContext = createContext();
const UserContext = createContext();
const NotificationContext = createContext();
```

---

### How do you optimize Context performance?

```jsx
// 1. Split contexts by update frequency
const ThemeContext = createContext();    // rarely changes
const UserContext = createContext();     // changes on login/logout
const CartContext = createContext();     // changes frequently

// 2. Memoize the value object
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Separate state and dispatch contexts
const StateContext = createContext();
const DispatchContext = createContext();

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Components that only dispatch don't re-render on state changes
function AddButton() {
  const dispatch = useContext(DispatchContext); // stable reference
  return <button onClick={() => dispatch({ type: "add" })}>Add</button>;
}
```

---

### How do you implement useContext + useReducer for global state?

```jsx
const initialState = {
  user: null,
  theme: "light",
  notifications: [],
};

function appReducer(state, action) {
  switch (action.type) {
    case "SET_USER":
      return { ...state, user: action.payload };
    case "TOGGLE_THEME":
      return { ...state, theme: state.theme === "light" ? "dark" : "light" };
    case "ADD_NOTIFICATION":
      return { ...state, notifications: [...state.notifications, action.payload] };
    case "CLEAR_NOTIFICATIONS":
      return { ...state, notifications: [] };
    default:
      return state;
  }
}

const AppStateContext = createContext();
const AppDispatchContext = createContext();

function AppProvider({ children }) {
  const [state, dispatch] = useReducer(appReducer, initialState);
  return (
    <AppDispatchContext.Provider value={dispatch}>
      <AppStateContext.Provider value={state}>
        {children}
      </AppStateContext.Provider>
    </AppDispatchContext.Provider>
  );
}

// Custom hooks for clean API
function useAppState() {
  const context = useContext(AppStateContext);
  if (!context) throw new Error("useAppState must be used within AppProvider");
  return context;
}

function useAppDispatch() {
  const context = useContext(AppDispatchContext);
  if (!context) throw new Error("useAppDispatch must be used within AppProvider");
  return context;
}

// Usage
function Header() {
  const { user, theme } = useAppState();
  const dispatch = useAppDispatch();
  return (
    <header className={theme}>
      <span>{user?.name}</span>
      <button onClick={() => dispatch({ type: "TOGGLE_THEME" })}>
        Toggle Theme
      </button>
    </header>
  );
}
```

---

## Redux Core Concepts

### What are the three principles of Redux?

1. **Single source of truth** — The entire app state lives in one store
2. **State is read-only** — The only way to change state is by dispatching an action
3. **Changes are made with pure functions** — Reducers are pure functions `(state, action) => newState`

```
View → dispatch(action) → Reducer → New State → View updates
```

```js
// Action
const increment = { type: "counter/increment", payload: 1 };

// Reducer (pure function)
function counterReducer(state = { value: 0 }, action) {
  switch (action.type) {
    case "counter/increment":
      return { value: state.value + action.payload };
    case "counter/decrement":
      return { value: state.value - action.payload };
    default:
      return state;
  }
}

// Store
const store = createStore(counterReducer);
store.dispatch(increment); // { value: 1 }
```

---

### What are actions, reducers, and the store?

| Concept | Role | Description |
|---|---|---|
| **Action** | What happened | Plain object with `type` and optional `payload` |
| **Reducer** | How state changes | Pure function: `(state, action) → newState` |
| **Store** | Where state lives | Single object holding the app state tree |
| **Dispatch** | Trigger change | `store.dispatch(action)` |
| **Selector** | Read state | Function to extract data from store |

```js
// Action creator
const addTodo = (text) => ({
  type: "todos/add",
  payload: { id: Date.now(), text, completed: false },
});

// Reducer
function todosReducer(state = [], action) {
  switch (action.type) {
    case "todos/add":
      return [...state, action.payload];
    case "todos/toggle":
      return state.map((todo) =>
        todo.id === action.payload ? { ...todo, completed: !todo.completed } : todo
      );
    default:
      return state;
  }
}

// Selector
const selectCompletedTodos = (state) =>
  state.todos.filter((todo) => todo.completed);
```

---

### What is combineReducers?

`combineReducers` splits reducers by domain and combines them into one root reducer.

```js
import { combineReducers } from "redux";

const rootReducer = combineReducers({
  todos: todosReducer,
  user: userReducer,
  ui: uiReducer,
});

// State shape:
// {
//   todos: [...],
//   user: { name: "Alice" },
//   ui: { sidebarOpen: true }
// }
```

---

### How do you connect Redux to React?

```jsx
import { Provider, useSelector, useDispatch } from "react-redux";

// Wrap app with Provider
function App() {
  return (
    <Provider store={store}>
      <TodoList />
    </Provider>
  );
}

// Use hooks in components
function TodoList() {
  const todos = useSelector((state) => state.todos);
  const dispatch = useDispatch();

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id} onClick={() => dispatch(toggleTodo(todo.id))}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**useSelector optimization:** React-Redux automatically subscribes and only re-renders when the selected value changes (referential equality check).

---

### What is the difference between useSelector and connect?

| Feature | `useSelector` (hooks) | `connect` (HOC) |
|---|---|---|
| Syntax | Hook inside component | HOC wrapping component |
| Memoization | Manual (Reselect or shallow compare) | `mapStateToProps` auto-shallow |
| Testing | Render the component | Can test `mapStateToProps` separately |
| TypeScript | Better type inference | More verbose types |
| Current status | **Recommended** | Legacy (still supported) |

```jsx
// Modern — useSelector
function Counter() {
  const count = useSelector((state) => state.counter.value);
  const dispatch = useDispatch();
  return <button onClick={() => dispatch(increment())}>{count}</button>;
}

// Legacy — connect
const mapStateToProps = (state) => ({ count: state.counter.value });
const mapDispatchToProps = { increment };
export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```

---

## Redux Toolkit

### What is Redux Toolkit (RTK) and why use it?

RTK is the **official, recommended** way to write Redux logic. It simplifies store setup, reduces boilerplate, and includes best practices by default.

| Feature | Vanilla Redux | Redux Toolkit |
|---|---|---|
| Store setup | `createStore` + middleware | `configureStore` (batteries included) |
| Reducers | Switch statements, immutable updates | `createSlice` with Immer |
| Async | Manual thunks | `createAsyncThunk` |
| API caching | Manual | RTK Query |
| DevTools | Manual setup | Automatic |
| Immutability | Manual spreading | Immer (write "mutations") |

---

### How does createSlice work?

`createSlice` generates action creators and action types automatically.

```js
import { createSlice } from "@reduxjs/toolkit";

const todosSlice = createSlice({
  name: "todos",
  initialState: [],
  reducers: {
    addTodo: {
      reducer: (state, action) => {
        state.push(action.payload); // Immer allows "mutations"
      },
      prepare: (text) => ({
        payload: { id: Date.now(), text, completed: false },
      }),
    },
    toggleTodo: (state, action) => {
      const todo = state.find((t) => t.id === action.payload);
      if (todo) todo.completed = !todo.completed;
    },
    removeTodo: (state, action) => {
      return state.filter((t) => t.id !== action.payload);
    },
  },
});

export const { addTodo, toggleTodo, removeTodo } = todosSlice.actions;
export default todosSlice.reducer;
```

---

### How does configureStore work?

```js
import { configureStore } from "@reduxjs/toolkit";
import todosReducer from "./todosSlice";
import userReducer from "./userSlice";

const store = configureStore({
  reducer: {
    todos: todosReducer,
    user: userReducer,
  },
  // Automatically includes: redux-thunk, dev tools, serializable check
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(logger),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

### What is createAsyncThunk?

`createAsyncThunk` generates thunks that dispatch `pending/fulfilled/rejected` actions.

```js
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

export const fetchUsers = createAsyncThunk(
  "users/fetchUsers",
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch("/api/users");
      if (!response.ok) throw new Error("Failed to fetch");
      return await response.json();
    } catch (err) {
      return rejectWithValue(err.message);
    }
  }
);

const usersSlice = createSlice({
  name: "users",
  initialState: { items: [], loading: false, error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  },
});

// Usage in component
function UserList() {
  const dispatch = useDispatch();
  const { items, loading, error } = useSelector((state) => state.users);

  useEffect(() => {
    dispatch(fetchUsers());
  }, [dispatch]);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <List items={items} />;
}
```

---

### What is RTK Query?

RTK Query is a powerful data fetching and caching tool built into Redux Toolkit.

```js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

const api = createApi({
  reducerPath: "api",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  tagTypes: ["User"],
  endpoints: (builder) => ({
    getUsers: builder.query({
      query: () => "/users",
      providesTags: ["User"],
    }),
    getUserById: builder.query({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: "User", id }],
    }),
    createUser: builder.mutation({
      query: (newUser) => ({
        url: "/users",
        method: "POST",
        body: newUser,
      }),
      invalidatesTags: ["User"],
    }),
  }),
});

export const { useGetUsersQuery, useGetUserByIdQuery, useCreateUserMutation } = api;

// Usage
function UserList() {
  const { data: users, isLoading, error } = useGetUsersQuery();
  const [createUser] = useCreateUserMutation();

  if (isLoading) return <Spinner />;
  return (
    <div>
      <button onClick={() => createUser({ name: "Alice" })}>Add</button>
      {users?.map((u) => <UserCard key={u.id} user={u} />)}
    </div>
  );
}
```

**Key features:** Automatic caching, cache invalidation with tags, polling, prefetching, optimistic updates, code generation.

---

## Redux Middleware & Async

### What is Redux middleware?

Middleware sits between dispatch and the reducer, enabling side effects, logging, and async operations.

```
dispatch(action) → Middleware 1 → Middleware 2 → Reducer → New State
```

```js
// Custom logging middleware
const logger = (store) => (next) => (action) => {
  console.log("Dispatching:", action);
  const result = next(action);
  console.log("Next state:", store.getState());
  return result;
};

// Custom analytics middleware
const analytics = (store) => (next) => (action) => {
  if (action.meta?.analytics) {
    trackEvent(action.meta.analytics);
  }
  return next(action);
};
```

---

### What is Redux Thunk?

Thunks are functions that delay dispatch, enabling async logic inside Redux.

```js
// Basic thunk (included by default in RTK)
function fetchUser(userId) {
  return async (dispatch, getState) => {
    dispatch(setLoading(true));
    try {
      const response = await fetch(`/api/users/${userId}`);
      const user = await response.json();
      dispatch(setUser(user));
    } catch (error) {
      dispatch(setError(error.message));
    } finally {
      dispatch(setLoading(false));
    }
  };
}

// dispatch
dispatch(fetchUser(123));
```

---

### What is Redux Saga?

Redux Saga uses generator functions to handle complex async flows.

```js
import { call, put, takeLatest, all } from "redux-saga/effects";

function* fetchUserSaga(action) {
  try {
    yield put({ type: "FETCH_USER_PENDING" });
    const user = yield call(fetch, `/api/users/${action.payload}`);
    const data = yield call([user, "json"]);
    yield put({ type: "FETCH_USER_SUCCESS", payload: data });
  } catch (error) {
    yield put({ type: "FETCH_USER_FAILURE", payload: error.message });
  }
}

function* watchFetchUser() {
  yield takeLatest("FETCH_USER_REQUEST", fetchUserSaga);
}

export default function* rootSaga() {
  yield all([watchFetchUser()]);
}
```

**Thunk vs Saga:**

| Feature | Thunk | Saga |
|---|---|---|
| Complexity | Simple | Complex |
| Learning curve | Low | High (generators) |
| Testing | Harder (mock dispatch) | Easier (declarative effects) |
| Use case | Simple async | Complex flows, race conditions, debounce |
| Cancellation | Manual | Built-in (`takeLatest`, `race`) |

---

## Zustand, Jotai & Recoil

### What is Zustand and when to use it?

Zustand is a minimal, hook-based state management library.

```js
import { create } from "zustand";

const useStore = create((set, get) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  // Async action
  fetchCount: async () => {
    const res = await fetch("/api/count");
    const data = await res.json();
    set({ count: data.count });
  },
}));

// Usage — auto-selects, only re-renders when count changes
function Counter() {
  const count = useStore((state) => state.count);
  const increment = useStore((state) => state.increment);
  return <button onClick={increment}>{count}</button>;
}
```

**Zustand vs Redux:**

| Feature | Zustand | Redux Toolkit |
|---|---|---|
| Boilerplate | Minimal | Moderate |
| Bundle size | ~1 KB | ~11 KB |
| DevTools | Plugin | Built-in |
| Middleware | Plugins | Built-in |
| Server state | Manual | RTK Query |
| Learning curve | Very low | Moderate |

---

### What is Jotai?

Jotai is an atomic state management library — each piece of state is an atom.

```js
import { atom, useAtom } from "jotai";

// Atoms
const countAtom = atom(0);
const doubleCountAtom = atom((get) => get(countAtom) * 2); // derived

// Usage
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const [doubleCount] = useAtom(doubleCountAtom);
  return (
    <div>
      <span>{count} (double: {doubleCount})</span>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
    </div>
  );
}
```

---

### What is Recoil?

Recoil (by Meta) introduces atoms and selectors for fine-grained state management.

```jsx
import { atom, selector, useRecoilState, useRecoilValue } from "recoil";

const todoListState = atom({
  key: "todoListState",
  default: [],
});

const filteredTodoListState = selector({
  key: "filteredTodoListState",
  get: ({ get }) => {
    const list = get(todoListState);
    const filter = get(todoListFilterState);
    switch (filter) {
      case "completed":
        return list.filter((t) => t.completed);
      case "uncompleted":
        return list.filter((t) => !t.completed);
      default:
        return list;
    }
  },
});

function TodoList() {
  const todos = useRecoilValue(filteredTodoListState);
  return todos.map((todo) => <TodoItem key={todo.id} item={todo} />);
}
```

---

### How do you compare state management libraries?

| Library | Paradigm | Bundle | Learning Curve | Best For |
|---|---|---|---|---|
| **Context + useReducer** | Built-in | 0 KB | Low | Simple shared state |
| **Redux Toolkit** | Flux / single store | ~11 KB | Moderate | Large apps, complex state |
| **Zustand** | Hook-based store | ~1 KB | Very low | Medium apps, simple API |
| **Jotai** | Atomic | ~3 KB | Low | Fine-grained reactivity |
| **Recoil** | Atomic (Meta) | ~20 KB | Moderate | Complex derived state |
| **MobX** | Observable / proxy | ~15 KB | Moderate | OOP-style reactivity |
| **Valtio** | Proxy-based | ~3 KB | Very low | Mutable-style API |

---

## State Architecture Patterns

### What is the Flux architecture?

```
Action → Dispatcher → Store → View → Action (unidirectional)
```

Redux is inspired by Flux but with a single store and pure reducers.

| Concept | Flux | Redux |
|---|---|---|
| Stores | Multiple | Single |
| Dispatcher | Central dispatcher | `store.dispatch()` |
| State mutation | Mutable in store | Immutable (reducers) |

---

### How do you structure Redux state for large apps?

```
src/
├── store/
│   ├── index.js              # configureStore
│   └── hooks.js              # typed useSelector/useDispatch
├── features/
│   ├── auth/
│   │   ├── authSlice.js
│   │   ├── authApi.js        # RTK Query endpoints
│   │   └── authSelectors.js
│   ├── todos/
│   │   ├── todosSlice.js
│   │   └── todosSelectors.js
│   └── ui/
│       └── uiSlice.js
└── services/
    └── api.js                # RTK Query base API
```

**Best practices:**
- Feature-based folder structure
- Co-locate slice, selectors, and API per feature
- Use Reselect for memoized selectors
- Normalize nested data with `createEntityAdapter`

---

### What is createEntityAdapter?

`createEntityAdapter` provides a standardized way to manage normalized collections.

```js
import { createEntityAdapter, createSlice } from "@reduxjs/toolkit";

const usersAdapter = createEntityAdapter({
  selectId: (user) => user.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

const usersSlice = createSlice({
  name: "users",
  initialState: usersAdapter.getInitialState({ loading: false }),
  reducers: {
    addUser: usersAdapter.addOne,
    updateUser: usersAdapter.updateOne,
    removeUser: usersAdapter.removeOne,
    setAllUsers: usersAdapter.setAll,
  },
});

// Generates memoized selectors
export const {
  selectAll: selectAllUsers,
  selectById: selectUserById,
  selectIds: selectUserIds,
} = usersAdapter.getSelectors((state) => state.users);
```

**Normalized state shape:**
```js
{
  ids: [1, 2, 3],
  entities: {
    1: { id: 1, name: "Alice" },
    2: { id: 2, name: "Bob" },
    3: { id: 3, name: "Charlie" }
  }
}
```

---

### What are Reselect selectors?

Reselect creates memoized selectors — they recompute only when inputs change.

```js
import { createSelector } from "reselect";

const selectTodos = (state) => state.todos;
const selectFilter = (state) => state.filter;

const selectFilteredTodos = createSelector(
  [selectTodos, selectFilter],
  (todos, filter) => {
    switch (filter) {
      case "completed":
        return todos.filter((t) => t.completed);
      case "active":
        return todos.filter((t) => !t.completed);
      default:
        return todos;
    }
  }
);

// Only recomputes when todos or filter change
const filtered = selectFilteredTodos(store.getState());
```

---

## Performance & Best Practices

### How do you avoid unnecessary re-renders with Redux?

```jsx
// ❌ Creates new object every render — always re-renders
const { user, theme } = useSelector((state) => ({
  user: state.user,
  theme: state.ui.theme,
}));

// ✅ Select individual values — re-renders only when that value changes
const user = useSelector((state) => state.user);
const theme = useSelector((state) => state.ui.theme);

// ✅ Or use shallowEqual for objects
import { shallowEqual } from "react-redux";
const { user, theme } = useSelector(
  (state) => ({ user: state.user, theme: state.ui.theme }),
  shallowEqual
);

// ✅ Use memoized selectors for derived data
const completedTodos = useSelector(selectCompletedTodos); // Reselect
```

---

### What are Redux best practices for senior developers?

1. **Use Redux Toolkit** — don't use vanilla Redux
2. **Normalize state** — use `createEntityAdapter` for collections
3. **Feature-based structure** — co-locate related logic
4. **Selectors for derived data** — use Reselect
5. **RTK Query for server state** — don't store API responses in slices
6. **Type your store** — `RootState`, `AppDispatch` types
7. **Keep reducers pure** — side effects in middleware/thunks
8. **Avoid storing derived data** — compute in selectors
9. **Use Immer** (built into RTK) — write readable "mutations"
10. **DevTools** — use Redux DevTools for debugging

```typescript
// Typed hooks (TypeScript)
import { TypedUseSelectorHook, useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";

export const useAppDispatch: () => AppDispatch = useDispatch;
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

---

### When should you choose Context vs Redux vs Zustand?

| Criteria | Context | Redux | Zustand |
|---|---|---|---|
| **Team size** | Small | Large | Any |
| **App complexity** | Low-medium | High | Medium |
| **Update frequency** | Low (theme, auth) | Any | Any |
| **DevTools** | React DevTools | Redux DevTools | Plugin |
| **Server state** | Pair with React Query | RTK Query | Pair with React Query |
| **Bundle concern** | 0 KB | ~11 KB | ~1 KB |
| **Testing** | Mock providers | Mock store | Simple |

**Decision tree:**
```
Is state only used by one component? → useState
Is state shared by nearby components? → Lift state up
Is state shared app-wide but rarely changes? → Context
Do you need middleware, DevTools, complex logic? → Redux Toolkit
Want minimal API with good DX? → Zustand
Need fine-grained atom-based reactivity? → Jotai
```

---

### How do you persist Redux state?

```js
import { configureStore } from "@reduxjs/toolkit";
import { persistStore, persistReducer } from "redux-persist";
import storage from "redux-persist/lib/storage"; // localStorage

const persistConfig = {
  key: "root",
  storage,
  whitelist: ["auth", "preferences"], // only persist these
  blacklist: ["ui"],                  // don't persist these
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ["persist/PERSIST", "persist/REHYDRATE"],
      },
    }),
});

const persistor = persistStore(store);

// Wrap app
<Provider store={store}>
  <PersistGate loading={<Spinner />} persistor={persistor}>
    <App />
  </PersistGate>
</Provider>
```

---

### How do you test Redux logic?

```js
// Test reducer
test("should add todo", () => {
  const state = todosReducer([], addTodo("Buy milk"));
  expect(state).toHaveLength(1);
  expect(state[0].text).toBe("Buy milk");
});

// Test selector
test("selectCompletedTodos", () => {
  const state = {
    todos: [
      { id: 1, text: "A", completed: true },
      { id: 2, text: "B", completed: false },
    ],
  };
  expect(selectCompletedTodos(state)).toHaveLength(1);
});

// Test async thunk
test("fetchUsers fulfilled", async () => {
  const store = configureStore({ reducer: { users: usersReducer } });
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve([{ id: 1, name: "Alice" }]),
  });
  await store.dispatch(fetchUsers());
  expect(store.getState().users.items).toHaveLength(1);
});

// Integration test with component
test("renders todos from store", () => {
  const store = configureStore({
    reducer: { todos: todosReducer },
    preloadedState: { todos: [{ id: 1, text: "Test", completed: false }] },
  });
  render(
    <Provider store={store}>
      <TodoList />
    </Provider>
  );
  expect(screen.getByText("Test")).toBeInTheDocument();
});
```

---

[← Back to README](./README.md)
