# JavaScript Fundamentals & Advanced

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Core Fundamentals](#core-fundamentals)
2. [Closures & Scope](#closures--scope)
3. [Prototypes & Inheritance](#prototypes--inheritance)
4. [Async JavaScript](#async-javascript)
5. [ES6+ Features](#es6-features)
6. [Performance & Memory](#performance--memory)
7. [Design Patterns](#design-patterns)
8. [Event Loop & Concurrency](#event-loop--concurrency)

---

## Core Fundamentals

### What are the primitive types in JavaScript?

JavaScript has **7 primitive types**:

| Type | Example | typeof |
|---|---|---|
| `string` | `"hello"` | `"string"` |
| `number` | `42`, `3.14`, `NaN`, `Infinity` | `"number"` |
| `bigint` | `9007199254740991n` | `"bigint"` |
| `boolean` | `true`, `false` | `"boolean"` |
| `undefined` | `undefined` | `"undefined"` |
| `null` | `null` | `"object"` ⚠️ (historical bug) |
| `symbol` | `Symbol("id")` | `"symbol"` |

Everything else is an **object** (arrays, functions, dates, etc.).

```js
typeof null === "object"        // true — historical JS bug
typeof function(){} === "function" // true — special case
typeof [] === "object"          // true — arrays are objects
Array.isArray([])               // true — correct check
```

---

### What is the difference between `==` and `===`?

- `===` (strict equality): checks value **and** type — no coercion
- `==` (loose equality): checks value after **type coercion**

```js
0 == false      // true  (false coerces to 0)
0 === false     // false (different types)
"" == false     // true
"" === false    // false
null == undefined  // true
null === undefined // false
NaN == NaN      // false (NaN is never equal to itself)
Object.is(NaN, NaN) // true — use Object.is for NaN comparison
```

**Rule:** Always use `===` unless you explicitly need coercion.

---

### What is `var`, `let`, and `const`?

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (initialized as `undefined`) | Yes (TDZ — not initialized) | Yes (TDZ) |
| Re-declare | Yes | No | No |
| Re-assign | Yes | Yes | No |
| Global property | Yes (`window.x`) | No | No |

```js
// var hoisting
console.log(x); // undefined (not ReferenceError)
var x = 5;

// let TDZ (Temporal Dead Zone)
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

// const — reference is constant, not the value
const obj = { a: 1 };
obj.a = 2;       // OK — mutating the object
obj = {};        // TypeError — cannot reassign const
```

---

### What is hoisting?

**Hoisting** is JavaScript's behavior of moving declarations to the top of their scope before execution.

```js
// What you write:
console.log(foo()); // "bar" — function declarations are fully hoisted
function foo() { return "bar"; }

// What the engine sees:
function foo() { return "bar"; }
console.log(foo());

// var hoisting — declaration only, not assignment
console.log(x); // undefined
var x = 5;
// Engine sees: var x; console.log(x); x = 5;

// let/const — TDZ: hoisted but not initialized
console.log(y); // ReferenceError
let y = 5;
```

---

### What is the Temporal Dead Zone (TDZ)?

The **TDZ** is the time between entering a block scope and the point where a `let` or `const` variable is declared. Accessing the variable in this period throws a `ReferenceError`.

```js
{
  // TDZ starts for x
  console.log(typeof x); // ReferenceError (not "undefined")
  let x = 5;             // TDZ ends
  console.log(x);        // 5
}
```

---

### What is type coercion?

Implicit conversion of a value from one type to another.

```js
// String conversion
"5" + 3      // "53" (number coerced to string)
"5" - 3      // 2 (string coerced to number)
"5" * "3"    // 15

// Boolean coercion (falsy values)
Boolean(0)         // false
Boolean("")        // false
Boolean(null)      // false
Boolean(undefined) // false
Boolean(NaN)       // false
Boolean(false)     // false
// Everything else is truthy

// Explicit coercion
Number("42")       // 42
Number("")         // 0
Number(null)       // 0
Number(undefined)  // NaN
String(42)         // "42"
Boolean(1)         // true
```

---

### What is `NaN` and how do you check for it?

`NaN` (Not a Number) results from invalid numeric operations:

```js
NaN === NaN           // false — NaN is not equal to itself!
typeof NaN            // "number" — confusing but true

// Correct checks:
Number.isNaN(NaN)     // true
Number.isNaN("hello") // false (does NOT coerce)
isNaN("hello")        // true (coerces first — avoid this)

// Pattern
const isNotANumber = (val) => Number.isNaN(val) || val !== val;
```

---

## Closures & Scope

### What is a closure?

A **closure** is a function that retains access to its lexical scope even when executed outside that scope.

```js
function makeCounter() {
  let count = 0;              // closed-over variable
  return {
    increment() { count++; },
    decrement() { count--; },
    value() { return count; }
  };
}

const counter = makeCounter();
counter.increment();
counter.increment();
console.log(counter.value()); // 2
// count is private — not accessible from outside
```

**Common use cases:**
- Data encapsulation / private variables
- Function factories
- Memoization
- Event handlers retaining state

```js
// Function factory
function multiply(x) {
  return (y) => x * y; // closes over x
}

const double = multiply(2);
const triple = multiply(3);
double(5); // 10
triple(5); // 15
```

---

### Classic closure loop bug and fix

```js
// Bug: all callbacks print 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 3, 3, 3
}

// Fix 1: use let (block scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 0, 1, 2
}

// Fix 2: IIFE to create new scope
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 100))(i);
}
```

---

### What is lexical scope?

**Lexical scope** means scope is determined by where variables are written in the source code, not where functions are called.

```js
const x = "global";

function outer() {
  const x = "outer";
  function inner() {
    console.log(x); // "outer" — resolved at write time, not call time
  }
  return inner;
}

const fn = outer();
f(); // "outer" — not "global"
```

---

### What is the scope chain?

When a variable is accessed, JS looks up the **scope chain** (from inner to outer) until found or until global scope is reached.

```js
const a = 1;

function outer() {
  const b = 2;
  function inner() {
    const c = 3;
    console.log(a, b, c); // 1, 2, 3 — full scope chain access
  }
  inner();
}
```

---

## Prototypes & Inheritance

### What is the prototype chain?

Every JavaScript object has a hidden `[[Prototype]]` link. Property lookup traverses this chain.

```js
const animal = {
  speak() { return `${this.name} makes a sound`; }
};

const dog = Object.create(animal);
dog.name = "Rex";
dog.bark = function() { return "Woof!"; };

dog.speak(); // "Rex makes a sound" — found on animal via prototype chain
dog.bark();  // "Woof!" — found directly on dog
dog.toString(); // found on Object.prototype
```

---

### What is the difference between `__proto__` and `prototype`?

- `__proto__`: the actual prototype link on every object instance
- `prototype`: the object attached to constructor functions, used to set `__proto__` on instances

```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  return `${this.name} speaks`;
};

const dog = new Animal("Rex");
dog.__proto__ === Animal.prototype     // true
Object.getPrototypeOf(dog) === Animal.prototype // true (preferred)
```

---

### Explain ES6 Classes and how they relate to prototypes

ES6 classes are **syntactic sugar** over prototype-based inheritance:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound`;
  }

  static create(name) {
    return new Animal(name);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);         // calls Animal constructor
    this.breed = breed;
  }

  speak() {
    return `${this.name} barks!`;
  }
}

const dog = new Dog("Rex", "Labrador");
dog.speak();                    // "Rex barks!"
dog instanceof Dog;             // true
dog instanceof Animal;          // true
Object.getPrototypeOf(Dog) === Animal; // true
```

---

### What are the ways to create objects in JavaScript?

```js
// 1. Object literal
const obj = { name: "Alice" };

// 2. Constructor function
function Person(name) { this.name = name; }
const p = new Person("Alice");

// 3. ES6 Class
class Person { constructor(name) { this.name = name; } }
const p2 = new Person("Alice");

// 4. Object.create()
const proto = { greet() { return `Hi, ${this.name}`; } };
const p3 = Object.create(proto);
p3.name = "Alice";

// 5. Object.assign()
const p4 = Object.assign({}, { name: "Alice" });

// 6. Spread
const p5 = { ...{ name: "Alice" } };
```

---

## Async JavaScript

### What is the Event Loop?

The event loop allows JavaScript (single-threaded) to perform non-blocking operations.

```
Call Stack → Web APIs → Callback Queue / Microtask Queue → Call Stack
```

**Priority:** Microtasks (Promises, queueMicrotask) > Macrotasks (setTimeout, setInterval, I/O)

```js
console.log("1");                         // sync

setTimeout(() => console.log("2"), 0);   // macrotask

Promise.resolve().then(() => console.log("3")); // microtask

console.log("4");                         // sync

// Output: 1, 4, 3, 2
```

---

### What are Promises?

A **Promise** represents a value that will be available in the future.

```js
// States: pending → fulfilled | rejected

const fetchUser = (id) =>
  new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) resolve({ id, name: "Alice" });
      else reject(new Error("Invalid ID"));
    }, 1000);
  });

// Chaining
fetchUser(1)
  .then(user => user.name)
  .then(name => console.log(name)) // "Alice"
  .catch(err => console.error(err))
  .finally(() => console.log("done"));

// Promise combinators
Promise.all([p1, p2, p3]);          // all resolve, or first rejection
Promise.allSettled([p1, p2, p3]);   // wait for all, regardless of result
Promise.race([p1, p2, p3]);         // first to settle wins
Promise.any([p1, p2, p3]);          // first to fulfill wins
```

---

### What is async/await?

**Async/await** is syntactic sugar over Promises, making async code look synchronous.

```js
async function getUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const user = await response.json();
    return user;
  } catch (error) {
    console.error("Failed to fetch user:", error);
    throw error;
  }
}

// Parallel execution (don't await sequentially if not dependent)
async function getMultiple() {
  // ❌ Sequential (slow — 2s total)
  const a = await fetchA(); // 1s
  const b = await fetchB(); // 1s

  // ✅ Parallel (fast — 1s total)
  const [a, b] = await Promise.all([fetchA(), fetchB()]);
}
```

---

### What is the difference between `setTimeout`, `setInterval`, and `requestAnimationFrame`?

```js
// setTimeout — executes once after delay
const id = setTimeout(() => console.log("once"), 1000);
clearTimeout(id);

// setInterval — repeats at interval
const id2 = setInterval(() => console.log("repeat"), 1000);
clearInterval(id2);

// requestAnimationFrame — syncs with browser repaint (~60fps)
function animate() {
  // update DOM
  requestAnimationFrame(animate); // ~16.7ms between calls
}
requestAnimationFrame(animate);
```

**Use rAF for animations** — it pauses when tab is hidden, preventing wasted work.

---

### What is a callback hell and how do you avoid it?

```js
// ❌ Callback hell (pyramid of doom)
getData(function(a) {
  getMoreData(a, function(b) {
    getEvenMore(b, function(c) {
      getYetMore(c, function(d) {
        console.log(d);
      });
    });
  });
});

// ✅ Promises (flat chain)
getData()
  .then(getMoreData)
  .then(getEvenMore)
  .then(getYetMore)
  .then(console.log)
  .catch(handleError);

// ✅ async/await (most readable)
async function process() {
  const a = await getData();
  const b = await getMoreData(a);
  const c = await getEvenMore(b);
  const d = await getYetMore(c);
  console.log(d);
}
```

---

## ES6+ Features

### What are destructuring, spread, and rest?

```js
// Array destructuring
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first=1, second=2, rest=[3,4,5]

// Object destructuring
const { name, age = 25, address: { city } } = user;

// Rename + default
const { name: userName = "Anonymous" } = user;

// Spread — expand
const arr2 = [...arr1, 4, 5];
const obj2 = { ...obj1, newProp: "value" };

// Rest parameters
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}

// Swap variables
let a = 1, b = 2;
[a, b] = [b, a];
```

---

### What are template literals and tagged templates?

```js
// Template literal
const greeting = `Hello, ${name}! You are ${age} years old.`;

// Multi-line
const html = `
  <div>
    <p>${text}</p>
  </div>
`;

// Tagged templates
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) =>
    result + str + (values[i] !== undefined ? `<mark>${values[i]}</mark>` : ""),
  "");
}

const name = "Alice";
const score = 95;
highlight`Hello ${name}, your score is ${score}!`;
// "Hello <mark>Alice</mark>, your score is <mark>95</mark>!"
```

---

### What are Symbols and when would you use them?

```js
// Symbols are unique and immutable
const id = Symbol("id");
const id2 = Symbol("id");
id === id2; // false — always unique

// Use case 1: unique object keys (no collisions)
const MY_KEY = Symbol("myKey");
obj[MY_KEY] = "value";

// Use case 2: well-known symbols to customize behavior
class MyArray {
  [Symbol.iterator]() {
    let index = 0;
    const data = [1, 2, 3];
    return {
      next() {
        return index < data.length
          ? { value: data[index++], done: false }
          : { done: true };
      }
    };
  }
}

for (const val of new MyArray()) {
  console.log(val); // 1, 2, 3
}
```

---

### What are Generators?

```js
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}

// Lazy iteration — values computed on demand
for (const num of range(0, 10, 2)) {
  console.log(num); // 0, 2, 4, 6, 8
}

// Infinite sequence (safe because lazy)
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
fib.next().value; // 0
fib.next().value; // 1
fib.next().value; // 1
fib.next().value; // 2
```

---

### What is optional chaining and nullish coalescing?

```js
// Optional chaining (?.) — short-circuits on null/undefined
const city = user?.address?.city;           // undefined (not error)
const len = arr?.length;
const result = obj?.method?.();             // safely call method

// Nullish coalescing (??) — fallback only for null/undefined
const name = user.name ?? "Anonymous";      // "Anonymous" if null or undefined
const count = 0 ?? 10;                      // 0 (not 10 — 0 is not nullish)
const empty = "" ?? "default";             // "" (not "default")

// vs OR operator (||) — fallback for ALL falsy values
0 || 10      // 10 (0 is falsy)
"" || "def"  // "def" (empty string is falsy)

// Nullish assignment (??=)
user.name ??= "Anonymous"; // assign only if null/undefined
```

---

### What are WeakMap and WeakSet?

```js
// WeakMap — keys must be objects, weakly held (GC eligible)
const cache = new WeakMap();

function process(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const result = expensiveComputation(obj);
  cache.set(obj, result);
  return result;
}
// When obj is GC'd, the WeakMap entry is automatically removed

// WeakSet — stores objects weakly
const visited = new WeakSet();

function markVisited(node) {
  visited.add(node);
}

function hasVisited(node) {
  return visited.has(node);
}
// No memory leaks — entries removed when nodes are GC'd
```

---

## Performance & Memory

### What are common memory leaks in JavaScript?

```js
// 1. Forgotten event listeners
element.addEventListener("click", handler);
// Fix:
element.removeEventListener("click", handler); // or use AbortController

// 2. Closures holding large data
function createHeavy() {
  const largeData = new Array(1000000).fill("data");
  return function() {
    return largeData.length; // holds reference to largeData
  };
}

// 3. Detached DOM nodes
let el = document.getElementById("btn");
document.body.removeChild(el);
// el still references the node — set to null when done
el = null;

// 4. Global variables accumulating state
window.data = []; // grows forever if never cleared

// 5. setInterval not cleared
const id = setInterval(work, 1000);
// Always clearInterval(id) on component unmount
```

---

### What is memoization?

Caching function results to avoid recomputation:

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensiveFn = memoize((n) => {
  // expensive computation
  return n * n;
});

expensiveFn(5); // computed
expensiveFn(5); // cached — instant
```

---

### What is debounce vs throttle?

```js
// Debounce — execute after delay from LAST call
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Use case: search input — only search after user stops typing
const handleSearch = debounce((query) => fetchResults(query), 300);

// Throttle — execute at most once per interval
function throttle(fn, interval) {
  let lastTime = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

// Use case: scroll handler — fire at most every 100ms
const handleScroll = throttle(() => updateUI(), 100);
```

---

## Design Patterns

### What is the Module Pattern?

```js
// IIFE-based module (pre-ES6)
const Counter = (() => {
  let count = 0; // private

  return {
    increment() { count++; },
    decrement() { count--; },
    getCount() { return count; }
  };
})();

// ES6 Modules
// math.js
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default function multiply(a, b) { return a * b; }

// app.js
import multiply, { PI, add } from "./math.js";
```

---

### What is the Observer Pattern?

```js
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, listener) {
    if (!this.events[event]) this.events[event] = [];
    this.events[event].push(listener);
    return this;
  }

  off(event, listener) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(l => l !== listener);
    }
    return this;
  }

  emit(event, ...args) {
    if (this.events[event]) {
      this.events[event].forEach(listener => listener(...args));
    }
    return this;
  }
}

const emitter = new EventEmitter();
emitter.on("data", (data) => console.log("received:", data));
emitter.emit("data", { id: 1 }); // "received: { id: 1 }"
```

---

### What is the Singleton Pattern?

```js
class Store {
  constructor() {
    if (Store.instance) return Store.instance;
    this.state = {};
    Store.instance = this;
  }

  set(key, value) { this.state[key] = value; }
  get(key) { return this.state[key]; }
}

const s1 = new Store();
const s2 = new Store();
s1 === s2; // true — same instance
```

---

## Event Loop & Concurrency

### What are microtasks vs macrotasks?

| Type | Examples | When run |
|---|---|---|
| **Microtasks** | `Promise.then`, `queueMicrotask`, `MutationObserver` | After every task, before next macrotask |
| **Macrotasks** | `setTimeout`, `setInterval`, `setImmediate`, I/O | One per event loop iteration |

```js
console.log("start");

setTimeout(() => console.log("setTimeout"), 0);       // macrotask

Promise.resolve()
  .then(() => console.log("promise 1"))               // microtask
  .then(() => console.log("promise 2"));              // microtask

queueMicrotask(() => console.log("queueMicrotask"));  // microtask

console.log("end");

// Output:
// start
// end
// promise 1
// queueMicrotask
// promise 2
// setTimeout
```

---

### What is Web Workers and when would you use them?

```js
// main.js — offload heavy computation to worker thread
const worker = new Worker("worker.js");

worker.postMessage({ data: largeArray });

worker.onmessage = (event) => {
  console.log("Result:", event.data.result);
};

worker.onerror = (error) => {
  console.error("Worker error:", error);
};

// worker.js — runs in separate thread (no DOM access)
self.onmessage = (event) => {
  const { data } = event.data;
  const result = data.reduce((sum, n) => sum + n, 0);
  self.postMessage({ result });
};
```

**Use cases:** Heavy computation, image processing, parsing large JSON, compression.

---

[← Back to README](./README.md)