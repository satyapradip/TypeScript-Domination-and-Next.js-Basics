# TypeScript-Domination-and-Next.js-Basics
# Module 1 — JavaScript Foundations for TypeScript & Next.js

> **Panchayat Dev Series** | Module 1 of 3
> Work through every section in order. Each section has: Theory → Example → Task → Quiz.

---

## Table of Contents

1. [Variables & Scope](#1-variables--scope)
2. [Functions & Arrow Functions](#2-functions--arrow-functions)
3. [Objects & Destructuring](#3-objects--destructuring)
4. [Arrays & Array Methods](#4-arrays--array-methods)
5. [Async / Await & Promises](#5-async--await--promises)
6. [Modules — import & export](#6-modules--import--export)
7. [Classes](#7-classes)
8. [Error Handling](#8-error-handling)
9. [Spread, Rest & Optional Chaining](#9-spread-rest--optional-chaining)
10. [Quick-fire Cheatsheet](#10-quick-fire-cheatsheet)

---

## 1. Variables & Scope

**Why this matters for TS:** TypeScript wraps around JS variables. Understanding `let`, `const`, and `var` is the foundation of how TS infers types. If you declare with `const`, TypeScript locks the type to the exact literal value — this is called *literal narrowing*, and it's very powerful.

### The Three Keywords

Never use `var` in modern JS/TS. It has function scope (confusing), is hoisted (dangerous), and can be re-declared (a bug trap). Use `let` for values that change and `const` for everything else.

```js
// NEVER do this
var name = "Strivion"; // function-scoped, hoisted, re-declarable

// USE THESE
let score = 0;                          // block-scoped, can reassign
const API_URL = "https://panchayat.dev"; // block-scoped, cannot reassign

score = 10;       // OK
// API_URL = "x"; // ERROR — const cannot be reassigned
```

### Block Scope

`let` and `const` live inside their `{ }` block. They don't leak out.

```js
if (true) {
  let msg = "hello";
  const x = 5;
}
// console.log(msg); // ERROR — msg is not defined outside the block
```

> **TS tip:** `const` narrows the type to the exact value. `const role = "admin"` gives type `"admin"`, not `string`. Very useful for union types later.

### Task 1

- Create a `const` for your Panchayat app name
- Create a `let` for current logged-in user (start as `null`, then reassign to a name string)
- Try putting a variable inside an `if` block and accessing it outside — observe the error
- Rewrite any old code you wrote with `var` using `let`/`const`

### Quiz

**Q: Which keyword should you use for a variable that never changes after being set?**
- [ ] var
- [ ] let
- [x] const ✅

---

## 2. Functions & Arrow Functions

**Why this matters for TS:** Every function in TypeScript has typed parameters and return types. You need to know the three function forms cold before you add types to them.

### Three Ways to Write a Function

```js
// 1. Function declaration — hoisted, can be called before definition
function greet(name) {
  return "Hello, " + name;
}

// 2. Function expression — NOT hoisted
const greet2 = function(name) {
  return "Hello, " + name;
};

// 3. Arrow function — shorter, no own `this`
const greet3 = (name) => {
  return "Hello, " + name;
};

// Arrow shorthand — implicit return when one expression
const greet4 = (name) => "Hello, " + name;
```

### Default Parameters

```js
const createComplaint = (title, status = "pending") => {
  return { title, status };
};

createComplaint("Road damage");          // { title: "Road damage", status: "pending" }
createComplaint("Water leak", "open");   // { title: "Water leak", status: "open" }
```

### The `this` Difference — Critical!

Arrow functions don't have their own `this`. They inherit it from the surrounding scope. Regular functions have their own `this`. This matters in React/Next.js class components and event handlers.

```js
const obj = {
  name: "Panchayat",
  regularFn: function() {
    return this.name; // "Panchayat" — `this` = obj
  },
  arrowFn: () => {
    return this.name; // undefined — arrow inherits outer `this`
  }
};
```

> **Tip:** In Next.js and React, you'll use arrow functions almost everywhere for callbacks, event handlers, and component functions. Get comfortable writing them fast.

### Task 2

- Write a function `formatComplaint(title, village, date)` that returns a formatted string
- Rewrite it as an arrow function
- Add a default parameter for date (use `new Date().toLocaleDateString()`)
- Write an `isAdmin(role)` arrow function that returns `true` if role is `"admin"`

---

## 3. Objects & Destructuring

**Why this matters for TS:** TypeScript's interfaces and types are literally shapes of objects. Master objects now and interfaces will feel natural.

### Object Basics & Shorthand

```js
const name = "Ramesh";
const village = "Nirsa";

// Old way
const user = { name: name, village: village };

// Shorthand — if key name equals variable name
const user2 = { name, village };

// Nested object
const complaint = {
  id: "C001",
  title: "No water supply",
  location: { district: "Dhanbad", pin: "826001" },
  createdAt: new Date()
};
```

### Destructuring Objects

Instead of accessing properties one by one, pull them out in one line. This is everywhere in Next.js (props, API responses, hooks).

```js
const { id, title, location } = complaint;
console.log(id);    // "C001"
console.log(title); // "No water supply"

// Rename while destructuring
const { title: complaintTitle } = complaint;

// Nested destructuring
const { location: { district } } = complaint;
console.log(district); // "Dhanbad"

// Default value while destructuring
const { status = "pending" } = complaint;
console.log(status); // "pending" (complaint has no status key)
```

### Destructuring in Function Parameters

React/Next.js components receive props as an object. You'll destructure them directly in the parameter.

```js
// Instead of this
function Card(props) {
  return props.title + " - " + props.village;
}

// Do this — destructure in the parameter itself
function Card({ title, village, status = "open" }) {
  return title + " - " + village + " [" + status + "]";
}
```

### Task 3

- Create a `complaint` object with: id, title, description, village, status, createdAt
- Destructure it into individual variables in one line
- Write a function `printComplaint({ title, village, status })` that logs a formatted summary
- Create a nested object for `user` with a `profile` inside — destructure the nested `profile.name`

### Quiz

**Q: What does `const { x = 10 } = {}` give you?**
- [ ] Error — x is not defined
- [x] `x = 10` (uses the default value) ✅
- [ ] `x = undefined`

---

## 4. Arrays & Array Methods

**Why this matters for TS:** In Panchayat you'll have arrays of complaints, users, schemes. TypeScript needs to know the type of items inside (`Complaint[]`). These methods are what you'll use every day in API routes and React components.

### The Big 5 Array Methods

```js
const complaints = [
  { id: 1, title: "Road damage", status: "open",   votes: 23 },
  { id: 2, title: "Water leak",  status: "closed", votes: 8  },
  { id: 3, title: "No power",    status: "open",   votes: 41 },
];

// .map() — transform each item, returns NEW array
const titles = complaints.map(c => c.title);
// ["Road damage", "Water leak", "No power"]

// .filter() — keep items that pass test, returns NEW array
const open = complaints.filter(c => c.status === "open");
// [{ id:1... }, { id:3... }]

// .find() — returns FIRST match (or undefined)
const highVotes = complaints.find(c => c.votes > 20);
// { id: 1, title: "Road damage", ... }

// .reduce() — collapse to single value
const totalVotes = complaints.reduce((sum, c) => sum + c.votes, 0);
// 72

// .some() / .every() — boolean checks
const hasOpen = complaints.some(c => c.status === "open");  // true
const allOpen = complaints.every(c => c.status === "open"); // false
```

### Array Destructuring

```js
const [first, second, ...rest] = complaints;
console.log(first.title); // "Road damage"
console.log(rest.length); // 1

// React useState returns an array — this is WHY you write it like this:
const [count, setCount] = useState(0);
//     ^value  ^setter    ^returns [value, setter]
```

> **Tip:** Chain methods together. In Next.js API routes you'll often do: `complaints.filter(...).map(...).sort(...)` — one clean, readable line.

### Task 4

- Create an array of 5 complaint objects (id, title, village, status, votes)
- Use `.filter()` to get only `"open"` complaints
- Use `.map()` to extract just the titles
- Use `.find()` to get a complaint by its id
- Use `.reduce()` to count total votes across all complaints
- Chain `.filter().map()` in one expression

---

## 5. Async / Await & Promises

**Why this matters for TS:** Every API call in Panchayat (MongoDB queries, Whisper transcription, Claude API) is async. TypeScript wraps this with `Promise<T>` — you need to know the underlying concept cold.

### The Async Model in 3 Steps

JavaScript runs one thing at a time (single-threaded). But async operations (network, file, DB) happen "in the background." Promises represent a value that will exist in the future.

```js
// A Promise is an object representing future completion or failure
const myPromise = new Promise((resolve, reject) => {
  const success = true;
  if (success) resolve("Done!");
  else reject("Something went wrong");
});

// Consuming with .then() / .catch() — old style, avoid in new code
myPromise.then(result => console.log(result)).catch(err => console.log(err));

// async/await — modern, use this always
async function fetchComplaint(id) {
  try {
    const response = await fetch(`/api/complaints/${id}`);
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Failed:", error);
    throw error; // re-throw so callers know it failed
  }
}

// await only works inside async functions
// In Next.js App Router server components you can use await at top level
```

### Running Multiple Async Tasks

```js
// Sequential — one after another (slower)
const user = await getUser(id);
const complaints = await getComplaints(id);

// Parallel — both at same time (faster)
const [user, complaints] = await Promise.all([
  getUser(id),
  getComplaints(id)
]);

// Promise.allSettled — don't fail if one rejects
const results = await Promise.allSettled([fetchA(), fetchB()]);
results.forEach(r => {
  if (r.status === "fulfilled") console.log(r.value);
  else console.log("Failed:", r.reason);
});
```

> **Warning:** Never use `await` inside a `.forEach()` — it doesn't work as expected. Use `for...of` for sequential async loops, or `Promise.all(array.map(...))` for parallel async loops.

### Task 5

- Write an async function `getSchemes()` that fetches from a public JSON API and returns the data
- Wrap it in try/catch and log a friendly error message on failure
- Write a function that uses `Promise.all` to fetch user + complaints in parallel
- In your Panchayat Next.js project, look at an API route — identify which calls are async

### Quiz

**Q: What happens if you `await` a rejected Promise without a try/catch?**
- [ ] The error is silently ignored
- [x] An unhandled promise rejection error is thrown ✅
- [ ] The function returns undefined

---

## 6. Modules — import & export

**Why this matters for TS:** TypeScript uses the exact same module system. Every `.ts` and `.tsx` file is a module. You'll be importing types, interfaces, utilities, and components constantly.

### Named vs Default Exports

```js
// --- utils/complaints.js ---

// Named export — can have many in one file
export const formatDate = (date) => new Date(date).toLocaleDateString("en-IN");
export const STATUSES = ["open", "closed", "pending"];

// Default export — only ONE per file, usually the main thing
export default function getComplaintById(id, complaints) {
  return complaints.find(c => c.id === id);
}

// --- pages/complaints.js ---

// Importing named exports — use curly braces, name must match exactly
import { formatDate, STATUSES } from "./utils/complaints";

// Importing default export — no braces, name is up to you
import getComplaintById from "./utils/complaints";

// Import both together
import getComplaintById, { formatDate } from "./utils/complaints";
```

> **Tip:** In Next.js, page components must be default exports. Utility functions, types, and constants should be named exports. You'll follow this pattern in every file of Panchayat.

### Task 6

- Create a `lib/utils.js` file with 3 named helper functions (date format, status label, truncate text)
- Import and use them in another file
- Create a constants file with named exports for your status options
- Identify: which files in your Panchayat project use default exports vs named exports?

---

## 7. Classes

**Why this matters for TS:** TypeScript adds access modifiers (`public`, `private`, `protected`) and typed properties to classes. Mongoose models in your Panchayat backend use this heavily.

### Class Anatomy

```js
class Complaint {
  id;
  title;
  status;

  // Constructor runs when you do `new Complaint(...)`
  constructor(id, title) {
    this.id = id;
    this.title = title;
    this.status = "pending";
    this.createdAt = new Date();
  }

  // Method
  close() {
    this.status = "closed";
  }

  // Getter — accessed like a property, not a function call
  get summary() {
    return `[${this.status.toUpperCase()}] ${this.title}`;
  }
}

const c = new Complaint("C001", "Road pothole");
console.log(c.summary); // "[PENDING] Road pothole"
c.close();
console.log(c.summary); // "[CLOSED] Road pothole"
```

### Inheritance

```js
class UrgentComplaint extends Complaint {
  constructor(id, title) {
    super(id, title); // call parent constructor first
    this.priority = "high";
  }

  escalate() {
    return `ESCALATED: ${this.title}`;
  }
}

const uc = new UrgentComplaint("C002", "Gas leak");
console.log(uc.priority);  // "high"
uc.close();                // inherits parent method
```

### Task 7

- Write a `User` class with name, email, role, and an `isAdmin()` method
- Write an `AdminUser` class that extends `User`
- Add a getter to format a display name
- Look at your Mongoose schemas — note how they're similar to class shapes

---

## 8. Error Handling

### try / catch / finally

```js
async function submitComplaint(data) {
  try {
    // Code that might throw
    const result = await db.complaints.create(data);
    return { success: true, data: result };
  } catch (error) {
    // Handle the error — error is an Error object
    console.error(error.message);
    return { success: false, error: error.message };
  } finally {
    // Always runs — use for cleanup (close connections, stop loading, etc.)
    console.log("submitComplaint finished");
  }
}

// Custom errors
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = "ValidationError";
  }
}

throw new ValidationError("Title is required");
```

> **Tip:** In Next.js API routes, always wrap DB calls in try/catch. Return proper HTTP status codes — 200 success, 400 bad request, 401 unauthorized, 404 not found, 500 server error.

---

## 9. Spread, Rest & Optional Chaining

**Why this matters for TS:** You'll use spread constantly in React state updates (immutability), rest params in flexible functions, and optional chaining when TypeScript can't guarantee a value exists.

### Spread Operator `...`

```js
// Spread arrays — copy or combine
const a = [1, 2, 3];
const b = [...a, 4, 5]; // [1, 2, 3, 4, 5]

// Spread objects — copy or override properties
const base    = { status: "open", votes: 0 };
const updated = { ...base, status: "closed" }; // { status: "closed", votes: 0 }

// React pattern: updating state immutably
setComplaint(prev => ({ ...prev, status: "closed" }));
```

### Rest Parameters

```js
// Collect remaining args into an array
function log(level, ...messages) {
  console.log(`[${level}]`, ...messages);
}
log("INFO", "User logged in", "at", new Date());

// Rest in destructuring
const [first, ...others] = [1, 2, 3, 4];
// first = 1, others = [2, 3, 4]

const { title, ...rest } = complaint;
// title = complaint.title, rest = everything else
```

### Optional Chaining `?.` and Nullish Coalescing `??`

```js
// Without optional chaining — dangerous
const city = user.address.city; // ERROR if address is null/undefined

// With optional chaining — safe
const city = user?.address?.city; // undefined if address doesn't exist

// Nullish coalescing — fallback for null or undefined only
const name = user?.name ?? "Anonymous";
// (different from || which also catches 0, "", false)
```

### Task 9 — Final Boss Task: Mini Panchayat

Combine everything you've learned:

- Create an array of complaint objects with realistic data
- Write an async function that "fetches" them (simulate with a `Promise` + `setTimeout`)
- Use `.filter().map()` to get only open complaints' titles
- Destructure in the map callback
- Use spread to "update" a complaint without mutating the original
- Use optional chaining when accessing nested location data
- Wrap everything in try/catch with a custom error class
- Export the function as a named export from a module

---

## 10. Quick-fire Cheatsheet

| Syntax | What it does |
|---|---|
| `const` / `let` | Block-scoped vars. Never use `var`. |
| `() => {}` | Arrow function. No own `this`. Use everywhere. |
| `{ a, b } = obj` | Object destructuring. Pull out props. |
| `[a, b] = arr` | Array destructuring. `useState` uses this. |
| `.map()` | Transform array → new array. |
| `.filter()` | Keep items that pass a test. |
| `.find()` | First match or `undefined`. |
| `.reduce()` | Collapse array to a single value. |
| `async` / `await` | Handle promises cleanly. |
| `try` / `catch` | Wrap all async and risky code. |
| `...` | Spread to copy/merge. Rest to collect args. |
| `?.` | Optional chaining. Safe nested access. |
| `??` | Nullish coalescing. Fallback for `null`/`undefined`. |
| `import` / `export` | Module system. Default vs named exports. |
| `class` | Blueprint for objects. TS adds types to this. |
| `Promise.all()` | Run multiple async tasks in parallel. |

---

## What's Next — Module 2: TypeScript

Once you've completed all 9 tasks, you're ready for **Module 2: TypeScript Fundamentals**. You'll learn:

- Primitive types, type inference, and type annotations
- Interfaces and type aliases — defining the shape of your Panchayat data models
- Union types, optional properties, and readonly
- Generics — writing flexible, reusable typed code
- Typing async functions with `Promise<T>`
- TypeScript + Next.js patterns (typed API routes, typed props, typed Mongoose models)

> Everything in this module was chosen specifically because TypeScript builds directly on top of it. If any section felt shaky, revisit it before moving on.

---

*Panchayat Dev Series — Module 1 of 3*
