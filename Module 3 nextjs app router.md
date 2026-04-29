# Module 3 — Next.js App Router for Panchayat

> **Panchayat Dev Series** | Module 3 of 3
> Prerequisites: Module 1 (JS Foundations) + Module 2 (TypeScript) complete.
> File conventions, server vs client components, data fetching, server actions, route handlers, auth middleware — all applied directly to the Panchayat complaint and scheme management system.

---

## Table of Contents

1. [App Router Mental Model](#1-app-router-mental-model)
2. [File Conventions](#2-file-conventions)
3. [Server vs Client Components](#3-server-vs-client-components)
4. [Data Fetching Patterns](#4-data-fetching-patterns)
5. [Route Handlers — API Endpoints](#5-route-handlers--api-endpoints)
6. [Server Actions](#6-server-actions)
7. [Dynamic Routes & Params](#7-dynamic-routes--params)
8. [Middleware & Auth](#8-middleware--auth)
9. [Panchayat Full Flow — Boss Task](#9-panchayat-full-flow--boss-task)
10. [Quick-fire Cheatsheet](#10-quick-fire-cheatsheet)

---

## 1. App Router Mental Model

The big shift: Next.js App Router (Next 13+) treats the filesystem as your routing layer and makes **React Server Components the default**. Everything runs on the server unless you explicitly opt into the client. This is fundamentally different from the old Pages Router.

### The two worlds

| | Server (default) | Client (`"use client"`) |
|---|---|---|
| Runs on | Node.js | Browser |
| DB access | Direct | Cannot — use API route |
| Secret env vars | Yes | No — never expose |
| JS sent to browser | Zero | Yes |
| useState / hooks | No | Yes |
| Event handlers / onClick | No | Yes |

> **Rule for Panchayat:** If a component fetches data or talks to MongoDB — server. If it has a button, form input, `useState`, or event handler — client. Keep client components small and at the leaf level. Most of your tree is server; only small interactive pieces at the bottom are client.

---

## 2. File Conventions

App Router uses special filenames inside the `app/` directory to create routes, layouts, loading states, and error boundaries automatically.

### Panchayat project structure

```
app/
  layout.tsx              // root layout — wraps every page
  page.tsx                // route: /
  loading.tsx             // auto shown while page fetches data
  error.tsx               // catches errors in this segment
  not-found.tsx           // custom 404
  complaints/
    page.tsx              // route: /complaints
    loading.tsx           // loading state for /complaints only
    [id]/
      page.tsx            // route: /complaints/C001
  complaints/new/
    page.tsx              // route: /complaints/new
  schemes/
    page.tsx              // route: /schemes
  admin/
    page.tsx              // route: /admin (protected)
  api/
    complaints/
      route.ts            // API: GET /api/complaints, POST /api/complaints
      [id]/
        route.ts          // API: GET/PATCH/DELETE /api/complaints/:id
        vote/
          route.ts        // API: POST /api/complaints/:id/vote
    voice/
      transcribe/
        route.ts          // API: POST /api/voice/transcribe (Whisper)
      categorize/
        route.ts          // API: POST /api/voice/categorize (Claude)
components/               // shared UI — NOT inside app/
lib/
  db.ts                   // MongoDB connection
  models/
    complaint.ts
    user.ts
  actions/
    complaints.ts         // server actions
types/
  index.ts                // your Module 2 interfaces live here
middleware.ts             // auth guard — at the ROOT of the project
```

### The 6 special filenames

| File | Purpose |
|---|---|
| `page.tsx` | Creates a route. Default export = the UI for that URL. |
| `layout.tsx` | Wraps child pages. Persists on navigation — doesn't remount. |
| `loading.tsx` | Automatic Suspense fallback while page data loads. |
| `error.tsx` | Error boundary for this route segment. Must be `"use client"`. |
| `not-found.tsx` | Custom 404 rendered when `notFound()` is called. |
| `route.ts` | API endpoint. No UI. Export HTTP verb functions (GET, POST, etc.). |

### Task 1

- Create `app/complaints/page.tsx` — list of all complaints
- Create `app/complaints/[id]/page.tsx` — single complaint detail
- Create `app/complaints/new/page.tsx` — submit a new complaint
- Create `app/schemes/page.tsx` — government schemes listing
- Add a `loading.tsx` to the complaints folder
- Create `app/layout.tsx` with a navbar and footer

---

## 3. Server vs Client Components

### Server component — default, no directive needed

```tsx
// app/complaints/page.tsx — SERVER component
// No "use client" → runs on server → can query DB directly

import { ComplaintModel } from "@/lib/models/complaint";
import { connectDB } from "@/lib/db";
import ComplaintCard from "@/components/ComplaintCard";

export default async function ComplaintsPage() {
  await connectDB();
  const complaints = await ComplaintModel.find({ status: "open" })
    .sort({ createdAt: -1 })
    .lean(); // .lean() returns plain JS objects, not Mongoose docs

  return (
    <main>
      <h1>Open Complaints</h1>
      {complaints.map(c => (
        <ComplaintCard key={c._id.toString()} complaint={c} />
      ))}
    </main>
  );
}
```

### Client component — `"use client"` at the very top

```tsx
"use client"; // must be the very first line

// components/VoteButton.tsx — CLIENT component
// Needs useState + onClick → must be client

import { useState } from "react";

interface VoteButtonProps {
  complaintId: string;
  initialVotes: number;
}

export default function VoteButton({ complaintId, initialVotes }: VoteButtonProps) {
  const [votes, setVotes]     = useState(initialVotes);
  const [loading, setLoading] = useState(false);

  async function handleVote() {
    setLoading(true);
    await fetch(`/api/complaints/${complaintId}/vote`, { method: "POST" });
    setVotes(v => v + 1);
    setLoading(false);
  }

  return (
    <button onClick={handleVote} disabled={loading}>
      {loading ? "Voting..." : `Vote (${votes})`}
    </button>
  );
}
```

> **Critical rule:** You CANNOT import a server component into a client component. But you CAN pass server components as `children` props into client components. Keep client components small, at the leaf level.

### Quiz

**Q: Which directive makes a Next.js component run in the browser?**
- [ ] `"use server"`
- [x] `"use client"` ✅
- [ ] No directive needed — all components are client by default

---

## 4. Data Fetching Patterns

App Router's superpower: server components are async functions. You can `await` database calls, API calls, and file reads right inside the component — no `useEffect` needed for initial data.

### Pattern 1 — direct DB in server component (best for Panchayat)

```tsx
// app/complaints/[id]/page.tsx
import { ComplaintModel } from "@/lib/models/complaint";
import { connectDB } from "@/lib/db";
import { notFound } from "next/navigation";

interface Props {
  params: { id: string };
}

export default async function ComplaintDetailPage({ params }: Props) {
  await connectDB();
  const complaint = await ComplaintModel.findById(params.id).lean();

  if (!complaint) notFound(); // renders not-found.tsx

  return (
    <div>
      <h1>{complaint.title}</h1>
      <p>{complaint.description}</p>
      <span>{complaint.status}</span>
    </div>
  );
}
```

### Pattern 2 — parallel data fetching (faster)

```tsx
// Fetch multiple things at the same time — don't await sequentially
export default async function DashboardPage() {
  const [complaints, schemes, userCount] = await Promise.all([
    getOpenComplaints(),
    getActiveSchemes(),
    getUserCount(),
  ]);

  return (
    <div>
      <StatsCard count={complaints.length} label="Open complaints" />
      <StatsCard count={schemes.length}    label="Active schemes" />
      <StatsCard count={userCount}         label="Registered citizens" />
    </div>
  );
}
```

### Pattern 3 — client-side fetching (for dynamic user interactions)

```tsx
"use client";
import { useState, useEffect } from "react";
import type { Complaint } from "@/types";

// Use this ONLY when you need to fetch after user interaction
// (search, filter, pagination, real-time updates)
export default function SearchComplaints() {
  const [query, setQuery]     = useState("");
  const [results, setResults] = useState<Complaint[]>([]);

  useEffect(() => {
    if (!query) return;
    fetch(`/api/complaints?q=${query}`)
      .then(r => r.json())
      .then(d => setResults(d.data));
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} placeholder="Search complaints..." />
      {results.map(c => <div key={c.id}>{c.title}</div>)}
    </div>
  );
}
```

> **Tip for Panchayat:** Fetch complaints list and scheme data in server components. Use client-side fetching only for the search/filter bar and the voice complaint recorder (which needs browser APIs like `MediaRecorder`).

### Task 2

- Make `app/complaints/page.tsx` fetch open complaints from MongoDB directly (server component)
- Make `app/complaints/[id]/page.tsx` fetch a single complaint and call `notFound()` if missing
- Add `Promise.all` to your dashboard page to fetch complaints + scheme counts in parallel
- Build a client-side search bar component that hits `/api/complaints?q=`

---

## 5. Route Handlers — API Endpoints

Route handlers live in `route.ts` files. You export named functions for each HTTP method: `GET`, `POST`, `PATCH`, `DELETE`. These are your Panchayat backend API endpoints.

### GET + POST on a collection

```ts
// app/api/complaints/route.ts
import { NextRequest, NextResponse } from "next/server";
import { connectDB } from "@/lib/db";
import { ComplaintModel } from "@/lib/models/complaint";
import type { NewComplaint } from "@/types";

export async function GET(req: NextRequest) {
  try {
    await connectDB();
    const { searchParams } = new URL(req.url);
    const status = searchParams.get("status") || "open";
    const q      = searchParams.get("q") || "";

    const filter: Record<string, unknown> = { status };
    if (q) filter.title = { $regex: q, $options: "i" };

    const complaints = await ComplaintModel.find(filter)
      .sort({ createdAt: -1 })
      .lean();

    return NextResponse.json({ success: true, data: complaints });
  } catch (error) {
    return NextResponse.json(
      { success: false, error: String(error) },
      { status: 500 }
    );
  }
}

export async function POST(req: NextRequest) {
  try {
    await connectDB();
    const body: NewComplaint = await req.json();

    if (!body.title || !body.village) {
      return NextResponse.json(
        { success: false, error: "title and village are required" },
        { status: 400 }
      );
    }

    const complaint = await ComplaintModel.create(body);
    return NextResponse.json({ success: true, data: complaint }, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { success: false, error: String(error) },
      { status: 500 }
    );
  }
}
```

### GET + PATCH + DELETE on a single item

```ts
// app/api/complaints/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { connectDB } from "@/lib/db";
import { ComplaintModel } from "@/lib/models/complaint";
import type { UpdateComplaint } from "@/types";

interface Context { params: { id: string } }

export async function GET(_req: NextRequest, { params }: Context) {
  await connectDB();
  const complaint = await ComplaintModel.findById(params.id).lean();
  if (!complaint) return NextResponse.json({ error: "Not found" }, { status: 404 });
  return NextResponse.json({ success: true, data: complaint });
}

export async function PATCH(req: NextRequest, { params }: Context) {
  await connectDB();
  const body: UpdateComplaint = await req.json();
  const updated = await ComplaintModel.findByIdAndUpdate(params.id, body, { new: true });
  return NextResponse.json({ success: true, data: updated });
}

export async function DELETE(_req: NextRequest, { params }: Context) {
  await connectDB();
  await ComplaintModel.findByIdAndDelete(params.id);
  return NextResponse.json({ success: true });
}
```

### Vote endpoint

```ts
// app/api/complaints/[id]/vote/route.ts
import { NextRequest, NextResponse } from "next/server";
import { connectDB } from "@/lib/db";
import { ComplaintModel } from "@/lib/models/complaint";

interface Context { params: { id: string } }

export async function POST(_req: NextRequest, { params }: Context) {
  try {
    await connectDB();
    await ComplaintModel.findByIdAndUpdate(params.id, { $inc: { votes: 1 } });
    return NextResponse.json({ success: true });
  } catch (error) {
    return NextResponse.json({ success: false, error: String(error) }, { status: 500 });
  }
}
```

### Task 3

- Build `app/api/complaints/route.ts` with GET (support `?status=` and `?q=`) and POST
- Build `app/api/complaints/[id]/route.ts` with GET, PATCH, DELETE
- Build `app/api/complaints/[id]/vote/route.ts` with POST that increments the votes field
- Test all routes with Postman or Thunder Client

### Quiz

**Q: In Next.js App Router, where do you put an API endpoint for `POST /api/complaints`?**
- [ ] `app/api/complaints/index.ts`
- [x] `app/api/complaints/route.ts` — export `async function POST` ✅
- [ ] `pages/api/complaints.ts`

---

## 6. Server Actions

Server actions let you run server-side code directly from a form or button click — without writing a separate API route. Perfect for Panchayat's complaint submission form. They use the `"use server"` directive.

### Server action defined inside a server component

```tsx
// app/complaints/new/page.tsx
import { redirect } from "next/navigation";
import { connectDB } from "@/lib/db";
import { ComplaintModel } from "@/lib/models/complaint";

async function submitComplaint(formData: FormData) {
  "use server"; // directive goes inside the function when defined in a server component

  const title       = formData.get("title") as string;
  const description = formData.get("description") as string;
  const village     = formData.get("village") as string;

  if (!title || !village) throw new Error("Title and village are required");

  await connectDB();
  await ComplaintModel.create({ title, description, village, status: "pending" });

  redirect("/complaints"); // redirect after success
}

export default function NewComplaintPage() {
  return (
    <form action={submitComplaint}> {/* pass the server action directly to action= */}
      <input    name="title"       placeholder="Complaint title"    required />
      <input    name="village"     placeholder="Your village"       required />
      <textarea name="description" placeholder="Describe the issue" />
      <button type="submit">Submit Complaint</button>
    </form>
  );
}
```

### Server actions in a dedicated file (reusable across pages)

```ts
// lib/actions/complaints.ts
"use server"; // at the top of the file → ALL exports are server actions

import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";
import { connectDB } from "@/lib/db";
import { ComplaintModel } from "@/lib/models/complaint";

export async function submitComplaint(formData: FormData) {
  const title   = formData.get("title") as string;
  const village = formData.get("village") as string;

  await connectDB();
  await ComplaintModel.create({ title, village, status: "pending" });

  revalidatePath("/complaints"); // bust the cache so the list updates immediately
  redirect("/complaints");
}

export async function closeComplaint(id: string) {
  await connectDB();
  await ComplaintModel.findByIdAndUpdate(id, { status: "closed" });
  revalidatePath("/complaints");
  revalidatePath(`/complaints/${id}`);
}

export async function deleteComplaint(id: string) {
  await connectDB();
  await ComplaintModel.findByIdAndDelete(id);
  revalidatePath("/complaints");
  redirect("/complaints");
}
```

> **When to use server actions vs route handlers:**
> - Use **server actions** for forms (complaint submission, scheme application, status updates from the admin panel).
> - Use **route handlers** when you need a public API that external clients can call — mobile app, the Whisper voice pipeline, the Claude categorization endpoint.
> - For Panchayat's voice pipeline specifically: Whisper transcription and Claude AI processing should be route handlers, not server actions.

---

## 7. Dynamic Routes & Params

```tsx
// [id] folder → params.id
// app/complaints/[id]/page.tsx

interface Props {
  params: { id: string };
}

export default async function ComplaintPage({ params }: Props) {
  const complaint = await getComplaint(params.id);
  return <h1>{complaint.title}</h1>;
}

// Generate static pages at build time (optional — for known IDs)
export async function generateStaticParams() {
  const complaints = await ComplaintModel.find().select("_id").lean();
  return complaints.map(c => ({ id: c._id.toString() }));
}

// Dynamic metadata for SEO — set <title> and <meta> per page
export async function generateMetadata({ params }: Props) {
  const complaint = await getComplaint(params.id);
  return {
    title:       `${complaint.title} | Panchayat`,
    description: complaint.description,
  };
}
```

### Catch-all routes

```
[id]          → /complaints/C001         → params.id = "C001"
[...slug]     → /blog/a/b/c             → params.slug = ["a","b","c"]
[[...slug]]   → /blog or /blog/a/b/c    → params.slug = [] or ["a","b","c"]
```

> **Tip:** Use `generateMetadata` on every detail page in Panchayat. It's how you set the page `<title>` and meta description dynamically — important for SEO on complaint and scheme detail pages.

---

## 8. Middleware & Auth

Middleware runs before every request is processed. In Panchayat, use it to protect admin routes — redirect to login if the user isn't authenticated.

```ts
// middleware.ts — at the ROOT of your project (not inside app/)
import { NextRequest, NextResponse } from "next/server";
import { getToken } from "next-auth/jwt"; // or your own JWT verification

export async function middleware(req: NextRequest) {
  const token        = await getToken({ req, secret: process.env.NEXTAUTH_SECRET });
  const isAdminRoute = req.nextUrl.pathname.startsWith("/admin");
  const isApiAdmin   = req.nextUrl.pathname.startsWith("/api/admin");

  if ((isAdminRoute || isApiAdmin) && !token) {
    // Not logged in — redirect to login
    return NextResponse.redirect(new URL("/login", req.url));
  }

  if ((isAdminRoute || isApiAdmin) && token?.role !== "admin") {
    // Logged in but not an admin — forbidden
    return NextResponse.redirect(new URL("/unauthorized", req.url));
  }

  return NextResponse.next(); // allow the request through
}

// Only run middleware on these paths — public routes are untouched
export const config = {
  matcher: ["/admin/:path*", "/api/admin/:path*"],
};
```

### Task 4

- Create `middleware.ts` at the project root that protects `/admin` routes
- Redirect unauthenticated users to `/login`
- Redirect non-admin users to `/unauthorized`
- Make sure public routes (`/complaints`, `/schemes`, `/api/complaints`) are not affected
- Test by visiting `/admin` while logged out

---

## 9. Panchayat Full Flow — Boss Task

This is where all three modules come together. Build the complete complaint submission and listing flow end-to-end.

### The 7-step complete flow

```
STEP 1 — types/index.ts (Module 2)
  ↓ Complaint interface, NewComplaint Omit, Status union, ApiResponse<T>

STEP 2 — lib/models/complaint.ts
  ↓ IComplaint extends Document, ComplaintSchema, ComplaintModel

STEP 3 — lib/actions/complaints.ts
  ↓ submitComplaint(), closeComplaint(), deleteComplaint() server actions

STEP 4 — app/complaints/new/page.tsx
  ↓ Server component with <form action={submitComplaint}>

STEP 5 — app/complaints/page.tsx
  ↓ Async server component, direct DB query, renders <ComplaintCard> list

STEP 6 — components/VoteButton.tsx
  ↓ "use client", useState, calls /api/complaints/:id/vote

STEP 7 — app/api/complaints/[id]/vote/route.ts
  ↓ POST handler, $inc votes, returns { success: true }
```

### Full code sketch

```ts
// types/index.ts — Module 2 interfaces
export interface Complaint {
  _id: string;
  title: string;
  description?: string;
  village: string;
  status: Status;
  votes: number;
  readonly createdAt: Date;
}
export type Status         = "open" | "closed" | "pending" | "escalated";
export type NewComplaint   = Omit<Complaint, "_id" | "createdAt" | "votes">;
export type UpdateComplaint = Partial<Complaint>;
export interface ApiResponse<T> { success: boolean; data: T; message?: string; }
```

```ts
// lib/models/complaint.ts
import { Schema, model, Document, models } from "mongoose";

interface IComplaint extends Document {
  title: string;
  description?: string;
  village: string;
  status: "open" | "closed" | "pending";
  votes: number;
}

const ComplaintSchema = new Schema<IComplaint>({
  title:       { type: String, required: true },
  description: { type: String },
  village:     { type: String, required: true },
  status:      { type: String, enum: ["open","closed","pending"], default: "pending" },
  votes:       { type: Number, default: 0 },
}, { timestamps: true });

export const ComplaintModel =
  models.Complaint || model<IComplaint>("Complaint", ComplaintSchema);
```

```tsx
// app/complaints/page.tsx — server component
import { ComplaintModel } from "@/lib/models/complaint";
import { connectDB } from "@/lib/db";
import ComplaintCard from "@/components/ComplaintCard";

export default async function ComplaintsPage() {
  await connectDB();
  const complaints = await ComplaintModel.find({ status: "open" })
    .sort({ createdAt: -1 })
    .lean();

  return (
    <main>
      <h1>Open Complaints ({complaints.length})</h1>
      <div>
        {complaints.map(c => (
          <ComplaintCard key={c._id.toString()} complaint={c} />
        ))}
      </div>
    </main>
  );
}
```

```tsx
// components/VoteButton.tsx — client component
"use client";
import { useState } from "react";

export default function VoteButton({ id, votes }: { id: string; votes: number }) {
  const [count, setCount]   = useState(votes);
  const [loading, setLoading] = useState(false);

  async function handleVote() {
    setLoading(true);
    await fetch(`/api/complaints/${id}/vote`, { method: "POST" });
    setCount(n => n + 1);
    setLoading(false);
  }

  return (
    <button onClick={handleVote} disabled={loading}>
      {loading ? "..." : `▲ ${count}`}
    </button>
  );
}
```

### Task 5 — Final Boss: complete Panchayat citizen flow

- Complete all 7 steps above end-to-end in your project
- Add `loading.tsx` to `app/complaints/` — show a skeleton card while data loads
- Add `error.tsx` to `app/complaints/` — show a friendly error with a retry button (must be `"use client"`)
- Add `generateMetadata` to the complaint detail page for SEO
- Wire up Whisper voice pipeline as a route handler at `app/api/voice/transcribe/route.ts`
- Wire up Claude AI categorization as a route handler at `app/api/voice/categorize/route.ts`
- Add middleware to protect `/admin` routes
- Test the complete flow: submit complaint → list → detail → vote

---

## 10. Quick-fire Cheatsheet

| Syntax / File | What it does |
|---|---|
| `page.tsx` | Creates a URL route. Default export = the page UI. |
| `layout.tsx` | Wraps all child pages. Persists on navigation. |
| `loading.tsx` | Automatic Suspense boundary while data loads. |
| `error.tsx` | Error boundary. Must be `"use client"`. |
| `route.ts` | API endpoint. Export `GET`, `POST`, `PATCH`, `DELETE`. |
| `"use client"` | Opt into browser. Enables hooks, events, and browser APIs. |
| `"use server"` | Marks a server action — callable from forms and buttons. |
| `async function Page()` | Server component. `await` DB directly inside it. |
| `params.id` | Dynamic route segment value from the `[id]` folder. |
| `notFound()` | Renders `not-found.tsx` for the current segment. |
| `redirect("/path")` | Server-side redirect after a mutation. |
| `revalidatePath("/path")` | Busts the Next.js cache for a path after mutation. |
| `middleware.ts` | Runs before every request. Place at project root. |
| `generateMetadata()` | Returns dynamic `<title>` and `<meta>` for a page. |
| `generateStaticParams()` | Pre-renders dynamic pages at build time. |
| `NextRequest` | Typed incoming request in route handlers. |
| `NextResponse` | Typed response builder. `.json()`, `.redirect()`. |
| `Promise.all([...])` | Fetch multiple data sources in parallel in server components. |
| `.lean()` | Mongoose — returns plain JS object, not a Mongoose document. |

---

## What's next for Panchayat

You've completed the full three-module series. Your next builds:

**Voice complaint pipeline** — `app/api/voice/transcribe/route.ts` receives audio, sends to OpenAI Whisper, returns transcript. `app/api/voice/categorize/route.ts` sends transcript to Claude, returns structured complaint data (title, village, category, priority).

**Authentication** — Wire up NextAuth.js with credentials provider. Add `citizen` and `admin` roles to the session. Protect `/admin` with middleware. Show/hide UI based on `session.user.role`.

**Admin panel** — Server components for the dashboard (complaint stats, pending count). Client components for status update dropdowns and bulk actions. Server actions for approve/reject/escalate.

**Real-time updates** — Use Next.js `revalidatePath` after each mutation so the complaints list stays fresh without a full page reload.

---

*Panchayat Dev Series — Module 3 of 3 — Series complete*
