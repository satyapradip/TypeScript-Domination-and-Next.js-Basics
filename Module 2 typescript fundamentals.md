# Module 2 — TypeScript Fundamentals

> **Panchayat Dev Series** | Module 2 of 3
> Prerequisite: Module 1 (JS Foundations) complete.
> TypeScript is JavaScript with a safety net. The compiler catches bugs before you run the code.

---

## Table of Contents

1. [Why TypeScript?](#1-why-typescript)
2. [Primitive Types](#2-primitive-types)
3. [Type Inference](#3-type-inference)
4. [Interfaces & Type Aliases](#4-interfaces--type-aliases)
5. [Union & Literal Types](#5-union--literal-types)
6. [Optional & Readonly Properties](#6-optional--readonly-properties)
7. [Functions with Types](#7-functions-with-types)
8. [Generics](#8-generics)
9. [Typing Async — Promise\<T\>](#9-typing-async--promiset)
10. [TypeScript + Next.js Patterns](#10-typescript--nextjs-patterns)
11. [Quick-fire Cheatsheet](#11-quick-fire-cheatsheet)

---

## 1. Why TypeScript?

TypeScript is JavaScript with type annotations. It compiles down to plain JS — browsers never see TS. The compiler catches bugs **before you run the code**. For a project like Panchayat with DB models, API routes, and AI integrations, this saves hours of debugging.

### JS vs TS — the core difference

```js
// JS — bug at runtime, no warning
function getTitle(c) {
  return c.tittle; // typo! No error until runtime
}
```

```ts
// TS — error at compile time
function getTitle(c: Complaint) {
  return c.tittle;
  // ERROR: Property 'tittle' does not exist on type 'Complaint'
}
```

> **How it works:** You write TypeScript, `tsc` (or the Next.js build) compiles it to JS. Your Panchayat project already has TS configured via `tsconfig.json`. Every `.ts` and `.tsx` file is TypeScript.

---

## 2. Primitive Types

The core idea: annotate variables with a type using `: TypeName` after the variable name.

```ts
// string
let appName: string = "Panchayat";

// number
let votes: number = 42;

// boolean
let isVerified: boolean = false;

// null and undefined — distinct types in TS
let userId: null = null;
let token: undefined = undefined;

// any — AVOID. Disables type checking. Last resort only.
let whatever: any = "could be anything";

// unknown — safer than any. Must check type before using.
let input: unknown = getUserInput();
if (typeof input === "string") {
  console.log(input.toUpperCase()); // now safe
}

// never — a function that NEVER returns (throws or infinite loops)
function crash(msg: string): never {
  throw new Error(msg);
}

// void — a function that returns nothing
function logComplaint(title: string): void {
  console.log(title);
}
```

> **Warning:** Never use `any` unless absolutely forced. It silently disables TypeScript for that variable. Use `unknown` when you genuinely don't know the type — it forces you to check before using.

### Task 1

- Annotate: app name (string), total complaints (number), dark mode toggle (boolean)
- Write a `logStatus` function with `: void` return type
- Try assigning a number to a string variable — read the TS error
- Replace any `any` you've written in Panchayat with `unknown` or a real type

### Quiz

**Q: Which type should you use when you don't know the type of an input but want to stay safe?**
- [ ] any
- [x] unknown ✅
- [ ] object

---

## 3. Type Inference

You don't always need to write types. TypeScript is smart enough to infer them from the value. Write annotations only when inference isn't enough.

```ts
// TS infers: string
const appName = "Panchayat";

// TS infers: number
const port = 3000;

// TS infers: boolean
const isDev = process.env.NODE_ENV === "development";

// TS infers: string[]
const statuses = ["open", "closed", "pending"];

// TS infers: { id: number; title: string }
const complaint = { id: 1, title: "No water" };

// Inference fails here — TS infers `any[]` for empty arrays
const items = [];           // bad — TS has no idea what will go in
const items: string[] = []; // good — tell TS what will go in

// Hover over any variable in VS Code to see its inferred type
```

> **Rule of thumb:** Let TS infer when you're initializing with a value. Write explicit types for function parameters, return types, and empty arrays/objects.

### Quiz

**Q: What type does TS infer for `const x = 5`?**
- [ ] any
- [x] number (specifically the literal `5`) ✅
- [ ] integer

---

## 4. Interfaces & Type Aliases

This is the heart of TypeScript. Interfaces define the shape of objects — exactly like your Mongoose schemas, but at the type level. In Panchayat, every DB model, API response, and component prop set gets an interface.

### `interface` — for object shapes

```ts
interface Complaint {
  id: string;
  title: string;
  description: string;
  village: string;
  status: "open" | "closed" | "pending"; // union literal — only these 3 values
  votes: number;
  createdAt: Date;
}

// Use it as a type annotation
const c: Complaint = {
  id: "C001",
  title: "Road damage",
  description: "Big pothole on NH-32",
  village: "Nirsa",
  status: "open",
  votes: 14,
  createdAt: new Date()
};

// Missing a field?  → TS errors immediately
// Wrong type for a field? → TS errors immediately
```

### `type` alias — flexible, for non-object types too

```ts
// Object shape (works like interface)
type User = {
  id: string;
  name: string;
  role: "citizen" | "admin" | "officer";
};

// Union type (can't do this with interface)
type Status = "open" | "closed" | "pending";

// Function type
type Handler = (req: Request) => Promise<Response>;

// Intersection — combine two types
type AdminUser = User & { permissions: string[] };
```

> **Convention:** Use `interface` for object shapes (especially Mongoose models and React props). Use `type` for unions, intersections, and function signatures. Either works for objects — be consistent within a file.

### Extending interfaces

```ts
interface BaseDocument {
  _id: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Complaint extends BaseDocument {
  title: string;
  status: Status;
}

interface User extends BaseDocument {
  name: string;
  email: string;
  role: "citizen" | "admin";
}
```

### Task 2

- Write a `Complaint` interface matching your Mongoose schema exactly
- Write a `User` interface with id, name, email, role
- Write a `GovScheme` interface for your scheme model
- Create a `BaseDocument` interface and extend all three from it
- Create a test object and intentionally break a field type — read the error

### Quiz

**Q: You want a type that is EITHER a string or a number. Which syntax is correct?**
- [ ] `type T = string & number`
- [x] `type T = string | number` ✅
- [ ] `interface T extends string, number`

---

## 5. Union & Literal Types

Complaint status, user roles, voice input languages — all have a fixed set of valid values. Union + literal types make TS enforce these exact values everywhere.

```ts
// Literal union — only these exact string values allowed
type Status   = "open" | "closed" | "pending" | "escalated";
type Role     = "citizen" | "admin" | "officer";
type Lang     = "hi" | "bn" | "en"; // Hindi, Bengali, English for Whisper

// Numeric literal union
type Priority = 1 | 2 | 3;

// Using them
function updateStatus(id: string, status: Status): void {
  // status can only be one of the 4 values above
  // passing "deleted" → TS error before runtime
}

// Discriminated union — powerful pattern for API responses
type ApiResponse =
  | { success: true;  data: Complaint }
  | { success: false; error: string };

function handleResponse(res: ApiResponse) {
  if (res.success) {
    console.log(res.data.title); // TS knows data exists here
  } else {
    console.log(res.error);      // TS knows error exists here
  }
}
```

> **Tip:** The discriminated union pattern (`success: true | false`) is perfect for your Panchayat API routes. Type the response shape and TS will catch every place you forget to handle the error case.

---

## 6. Optional & Readonly Properties

```ts
interface Complaint {
  id: string;
  title: string;
  description?: string;    // optional — may or may not exist
  resolvedAt?: Date;       // optional — only set when status = "closed"
  readonly _id: string;    // readonly — cannot be changed after creation
  readonly createdAt: Date;
}

// Accessing optional properties safely
function showResolved(c: Complaint) {
  // Can't just do c.resolvedAt.toISOString() — might be undefined
  if (c.resolvedAt) {
    console.log(c.resolvedAt.toISOString()); // safe inside the check
  }
  // Or use optional chaining
  console.log(c.resolvedAt?.toISOString());
}

// readonly array
const STATUSES: readonly string[] = ["open", "closed", "pending"];
// STATUSES.push("new") → ERROR
```

> **Tip:** In Panchayat, `_id` and `createdAt` from MongoDB should always be `readonly`. Fields like `resolvedAt` and `assignedTo` should be optional (`?`).

---

## 7. Functions with Types

Typed functions are where your JS knowledge pays off directly. You add types to parameters and return values — everything else is the same.

```ts
// Parameter types + explicit return type
function formatDate(date: Date): string {
  return date.toLocaleDateString("en-IN");
}

// Arrow function with types
const isAdmin = (role: Role): boolean => role === "admin";

// Optional parameter
function greet(name: string, title?: string): string {
  return title ? `${title} ${name}` : name;
}

// Default parameter with type
function createComplaint(title: string, status: Status = "pending"): Complaint {
  return { id: crypto.randomUUID(), title, status, createdAt: new Date() };
}

// Destructured object parameter with type
function printUser({ name, role }: User): void {
  console.log(`${name} (${role})`);
}

// Function that accepts a typed callback
function processComplaints(
  complaints: Complaint[],
  callback: (c: Complaint) => string
): string[] {
  return complaints.map(callback);
}
```

### Task 3

- Add types to your `formatComplaint`, `isAdmin`, and `createComplaint` functions from Module 1
- Write a typed `filterByStatus(complaints: Complaint[], status: Status): Complaint[]`
- Write a typed function that takes a callback parameter
- Intentionally pass the wrong type to a function — observe the error

---

## 8. Generics

Generics let you write reusable, type-safe code. Think of `<T>` as a placeholder for a type that gets filled in when you use the function. This is how React, Mongoose, and the Fetch API are all typed internally.

```ts
// Without generics — loses type info
function first(arr: any[]): any {
  return arr[0]; // returns any — useless to the caller
}

// With generics — T is filled in at call time
function first<T>(arr: T[]): T {
  return arr[0];
}

const title = first(["Road", "Water", "Power"]); // T = string → returns string
const vote  = first([42, 7, 11]);                // T = number → returns number

// Generic interface — API response wrapper
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
}

// Use it for specific shapes
type ComplaintResponse  = ApiResponse<Complaint>;
type UserResponse       = ApiResponse<User>;
type ListResponse<T>    = ApiResponse<T[]>;

// Generic with constraint — T must have an id field
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

findById(complaints, "C001"); // works — Complaint has id
findById(users, "U001");      // works — User has id
```

> **Tip:** You'll use generics every day in Panchayat: `useState<Complaint[]>([])`, `useRef<HTMLInputElement>(null)`, `Model<IComplaint>` in Mongoose. You don't need to *write* generics often — but you absolutely need to *read* them.

### Quiz

**Q: What does `useState<string>("")` tell TypeScript?**
- [ ] The initial value is a generic string
- [x] The state variable will always hold a string type ✅
- [ ] It creates a new generic component

---

## 9. Typing Async — Promise\<T\>

Every async function returns a Promise. TypeScript lets you specify what the Promise resolves to using `Promise<T>`. This means callers know exactly what they'll get back — no guessing.

```ts
// async function return type
async function fetchComplaint(id: string): Promise<Complaint> {
  const res  = await fetch(`/api/complaints/${id}`);
  const data = await res.json();
  return data as Complaint;
}

// Return null on failure
async function getUser(id: string): Promise<User | null> {
  try {
    const user = await UserModel.findById(id);
    return user;
  } catch {
    return null;
  }
}

// Using the ApiResponse generic pattern
async function submitComplaint(
  data: Omit<Complaint, "_id" | "createdAt">
): Promise<ApiResponse<Complaint>> {
  try {
    const complaint = await ComplaintModel.create(data);
    return { success: true, data: complaint };
  } catch (error) {
    return { success: false, data: null, message: String(error) };
  }
}
```

### Utility Types — TS built-ins you'll use every day

```ts
// Omit — remove specific fields from an interface
type NewComplaint    = Omit<Complaint, "_id" | "createdAt" | "votes">;

// Partial — make all fields optional (for PATCH/update routes)
type UpdateComplaint = Partial<Complaint>;

// Pick — keep only specific fields
type ComplaintCard   = Pick<Complaint, "id" | "title" | "status" | "votes">;

// Required — make all optional fields required
type FullComplaint   = Required<Complaint>;

// Record — object with typed keys and values
type StatusCount     = Record<Status, number>;
// { open: number; closed: number; pending: number; escalated: number }
```

> **Tip:** `Omit` and `Partial` are your two most-used utility types in Panchayat. Use `Omit` when creating new documents (exclude `_id`, `createdAt`), and `Partial` for update/PATCH API routes.

### Task 4

- Add `Promise<T>` return types to your Panchayat API fetch functions
- Create a `NewComplaint` type using `Omit` that removes `_id` and `createdAt`
- Create an `UpdateComplaint` type using `Partial`
- Write a typed `ApiResponse<T>` interface and use it in one API route

---

## 10. TypeScript + Next.js Patterns

This is where everything comes together for Panchayat. These are the exact patterns you'll use repeatedly in your Next.js App Router codebase.

### Typed React component props

```tsx
interface ComplaintCardProps {
  complaint: Complaint;
  onVote: (id: string) => void;
  showActions?: boolean; // optional
}

export default function ComplaintCard({
  complaint,
  onVote,
  showActions = true
}: ComplaintCardProps) {
  return (
    <div>
      <h2>{complaint.title}</h2>
      {showActions && (
        <button onClick={() => onVote(complaint.id)}>Vote</button>
      )}
    </div>
  );
}
```

### Typed Next.js API route (App Router)

```ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest): Promise<NextResponse> {
  try {
    const body: NewComplaint = await req.json();
    const complaint = await ComplaintModel.create(body);
    return NextResponse.json({ success: true, data: complaint }, { status: 201 });
  } catch (error) {
    return NextResponse.json({ success: false, error: String(error) }, { status: 500 });
  }
}
```

### Typed useState and useRef

```tsx
const [complaints, setComplaints] = useState<Complaint[]>([]);
const [user, setUser]             = useState<User | null>(null);
const [status, setStatus]         = useState<Status>("pending");

const inputRef = useRef<HTMLInputElement>(null);
const formRef  = useRef<HTMLFormElement>(null);
```

### Mongoose model with TypeScript

```ts
import { Schema, model, Document } from "mongoose";

// 1. Define the interface
interface IComplaint extends Document {
  title: string;
  description: string;
  village: string;
  status: "open" | "closed" | "pending";
  votes: number;
  createdAt: Date;
}

// 2. Define the schema
const ComplaintSchema = new Schema<IComplaint>({
  title:       { type: String, required: true },
  description: { type: String },
  village:     { type: String, required: true },
  status:      { type: String, enum: ["open", "closed", "pending"], default: "pending" },
  votes:       { type: Number, default: 0 },
}, { timestamps: true });

// 3. Export the typed model
export const ComplaintModel = model<IComplaint>("Complaint", ComplaintSchema);
```

### Task 5 — Boss Task: Type the entire Panchayat core

- Create a `types/index.ts` file and define interfaces for `Complaint`, `User`, `GovScheme`, `ApiResponse<T>`
- Add `Omit`/`Partial` utility types for create and update operations
- Type your existing Mongoose models with `IComplaint extends Document`
- Add prop types to at least one existing React component
- Add `Promise<NextResponse>` return types to your API routes
- Add `useState<T>` generics to all your state variables

---

## 11. Quick-fire Cheatsheet

| Syntax | What it does |
|---|---|
| `: string` | Primitive type annotation |
| `unknown` | Safer than `any` — must check type first |
| `interface` | Object shape definition. Use `extends` to inherit. |
| `type` | Type alias. Use for unions and function types. |
| `A \| B` | Union — either A or B |
| `A & B` | Intersection — must satisfy both A and B |
| `prop?` | Optional property — may or may not exist |
| `readonly` | Cannot reassign after creation |
| `<T>` | Generic — type placeholder filled in at call site |
| `Promise<T>` | Return type of async functions |
| `Omit<T, K>` | Remove specific fields from a type |
| `Partial<T>` | All fields become optional |
| `Pick<T, K>` | Keep only specified fields |
| `Record<K, V>` | Object with typed keys and values |
| `as Type` | Type assertion — use sparingly |
| `extends Document` | Mongoose model interface base |

---

## What's Next — Module 3: Next.js App Router

Once you've completed all 5 tasks (especially the Boss Task), you're ready for **Module 3: Next.js App Router Patterns for Panchayat**. You'll learn:

- App Router file conventions: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`
- Server vs Client components — when to use each
- Server Actions — typed form submissions without API routes
- Route handlers — typed `NextRequest` / `NextResponse`
- Data fetching patterns: server-side, client-side, and hybrid
- Middleware for authentication
- Building the Panchayat complaint submission and dashboard flows end-to-end

> Every pattern in this module uses the TypeScript interfaces and utility types you defined in Module 2. The two modules are designed to stack — your `types/index.ts` file from this module becomes the backbone of Module 3.

---

*Panchayat Dev Series — Module 2 of 3*
