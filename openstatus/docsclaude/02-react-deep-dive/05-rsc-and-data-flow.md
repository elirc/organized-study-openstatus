# 05 — Server Components, Hydration, and Data Flow in Next.js

Modern Next.js (App Router) lets components render on the server, on the client, or both. The boundary is the single most-misunderstood part of the framework. This file gives you the model that makes it click.

## Two component kinds, one tree

In the App Router, every file in `app/` is a **server component** by default. Adding `"use client"` at the top makes it (and every component it imports) a **client component**.

| | Server component | Client component |
| --- | --- | --- |
| Where it runs | Server only (per request, or build time for static) | Server (for SSR markup) **and** browser |
| Can use hooks? | No (`useState`, `useEffect`, etc.) | Yes |
| Can do async I/O? | Yes — `await` directly in component body | No, only inside `useEffect` or via React Query |
| Can access DB / env? | Yes | No (anything in the bundle ships to the browser) |
| Can use event handlers? | No (no JS in the browser) | Yes |
| Bundled and shipped to browser? | No (just the rendered HTML) | Yes |

The killer feature: a server component can `import` a client component and render it inside its tree. The reverse — a client component importing a server component — is not allowed (because the import would pull server code into the bundle). But a client component can render server components passed in as `children`.

## What `"use client"` actually means

It's a **boundary marker**, not an annotation. It says "this module and everything reached only from here is a client module." React/Next.js use the marker to:

1. Bundle the marked module (and its deps) for the browser.
2. Render the marked component on the server *as well*, to produce SSR HTML.
3. Hydrate the marked component in the browser (attach event listeners, reconcile against the SSR HTML, take over).

The marker applies to the file. If a server component file imports a `"use client"` file, the server file *stays* a server component — only the imported part is client.

## A real OpenStatus example

[`apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx) is a **server component**. It can `await` data:

```tsx
export default async function Layout({ children, params }: Props) {
  const { id } = await params;
  await fetchQueryOrNotFound(
    trpc.page.get.queryOptions({ id: Number.parseInt(id) }),
  );
  return <HydrateClient>{/* ... */}{children}</HydrateClient>;
}
```

It awaits a tRPC query *server-side*, populating React Query's cache. `HydrateClient` ships that cache state to the browser.

[`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx) is a **client component** (`"use client"` at the top). It uses hooks (`useQuery`, `useMutation`, `useParams`, `useTRPC`).

When the user navigates here, the server-side `Layout` runs, prefetches the page data, and renders `HydrateClient` around the page. The client `Page` mounts in the browser, calls `useQuery`, and finds the data **already in cache** — no network round-trip. The user sees the page populated immediately.

This is the most powerful pattern in App Router + React Query: **server-side prefetch into a client cache**.

## Server actions — mutations from forms without an API

A server action is a function marked `"use server"` that the client can invoke directly. Next.js generates an internal endpoint, serializes the args, runs the function on the server, returns the result.

```ts
"use server";
export async function createPost(formData: FormData) {
  const title = formData.get("title");
  await db.insert(...);
  revalidatePath("/posts");
}
```

```tsx
<form action={createPost}>
  <input name="title" />
  <button>Submit</button>
</form>
```

OpenStatus doesn't lean on server actions — it uses tRPC mutations instead. The reason: tRPC gives you typed clients across two different surfaces (dashboard + status-page) plus background workflows, with one API definition. Server actions are great for self-contained Next apps; tRPC is great for shared APIs.

## Hydration — the bridge between server and client

Hydration is what happens after the browser receives server-rendered HTML:

1. React mounts the component tree in the browser.
2. It walks the existing DOM and attaches event handlers in place.
3. The page becomes interactive.

Hydration **must produce the same tree the server produced**. If the server rendered "Logged in as Alice" and the client renders "Loading…", React warns. Common causes:

- Using `Date.now()`, `Math.random()`, `window.innerWidth` in render → server and client see different values.
- Reading `localStorage` or `cookies` directly during render in a client component.
- Time zones — server in UTC, browser in local.

Fix by deferring browser-only render to after mount:

```jsx
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return null;   // or a placeholder matching the SSR output
```

Or, mark the component with `"use client"` and dynamically import with `{ ssr: false }` so it never renders server-side.

## Streaming and Suspense

App Router supports streaming HTML out of order. Wrap slow parts in `<Suspense fallback={...}>`; the rest of the page sends immediately, the suspense boundary streams when the data resolves.

```tsx
export default function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <SlowList />   // can await inside
      </Suspense>
    </>
  );
}
```

OpenStatus uses `Suspense` sparingly; React Query's loading states cover most cases for its client-heavy dashboards.

## Cookies, sessions, and the request-scoped world

Server components run **per request**. They have access to cookies, headers, and the URL via Next's `cookies()`, `headers()` functions (from `next/headers`). They **cannot** be cached across requests for the same user without care — the request-scoped data would leak.

OpenStatus's server-side tRPC client ([`apps/dashboard/src/lib/trpc/server.tsx:38-53`](../../apps/dashboard/src/lib/trpc/server.tsx#L38-L53)) explicitly forwards cookies on every fetch so authenticated server requests carry the user's session:

```ts
fetch: async (url, options) => {
  const cookieStore = await cookies();
  return fetch(url, {
    ...options,
    credentials: "include",
    headers: { ...options?.headers, cookie: cookieStore.toString() },
  });
}
```

That's why a server-rendered page sees the same workspace the logged-in user does in the browser.

## When to use a server component vs a client component

| Use a server component when… | Use a client component when… |
| --- | --- |
| Reading data from a DB or API | You need state or effects |
| The component shouldn't ship JS | You handle user events (clicks, input) |
| You can compute everything once and ship HTML | You use a hook (`useState`, `useQuery`, etc.) |
| You want fast first paint | You integrate with browser APIs |

Default to server. Reach for client only when you need a hook, event handler, or browser API. This minimizes the bundle and maximizes performance.

## Static vs dynamic rendering

By default, a server component is **static** — rendered once at build time, served from CDN. Adding any "dynamic" signal (reading cookies, `headers()`, `searchParams`, calling `revalidatePath`) flips it to **dynamic** — rendered per request.

You can force one or the other:

```ts
export const dynamic = "force-dynamic";   // always per-request
export const dynamic = "force-static";    // always at build
export const revalidate = 60;             // ISR: re-render every 60s max
```

OpenStatus's public status page is intentionally dynamic (data freshness matters). The dashboard is dynamic by virtue of using cookies and auth.

## Hydration mismatches — the classic bug

You'll see one. The console message looks like:

> Hydration failed because the initial UI does not match what was rendered on the server.

Triage:

1. Is anything time-sensitive in render? (`new Date()`, `Date.now()`)
2. Is anything random in render? (`Math.random()`)
3. Is the component reading the browser environment? (`window`, `localStorage`)
4. Is a parent using a different layout on server vs client? (CSS `display: none` based on JS state)

Move the offender to `useEffect`. Defer with `mounted` gate if needed.

## Pitfall checklist

1. **Importing a server-only module from a client component.** Bundles the server code (env vars, secrets, DB clients). Use `import "server-only"` to throw at import time.
2. **Forgetting `"use client"`** on a component using hooks. Cryptic build errors.
3. **`async` in a client component body.** Only server components can be async. Move data fetching to React Query / useEffect.
4. **Reading cookies in a client component.** Use a server component to read them and pass values down.
5. **Hydration mismatch from `new Date()` or `Math.random()`.** Move to effect.
6. **Server actions inside loops without batching.** Each invocation is a round-trip.
7. **Not awaiting `params` in a server component.** Next.js 15 made these async; you'll get `undefined` if you forget.
8. **Caching dynamic data accidentally.** Per-user data in a static page.
9. **Mixing tRPC server proxy with client proxy** in the same component. Pick one based on the component's runtime.
10. **Suspense fallback that doesn't match the final layout.** Causes layout shift; users notice.

## Drills

- Find every `"use client"` in [`apps/dashboard/src/app/`](../../apps/dashboard/src/app/). Identify *why* each one is client (which hook or event made it client?). Could any become server again?
- Read [`apps/dashboard/src/lib/trpc/server.tsx`](../../apps/dashboard/src/lib/trpc/server.tsx) and [`apps/dashboard/src/lib/trpc/client.tsx`](../../apps/dashboard/src/lib/trpc/client.tsx) side by side. Identify the differences. Why can't they be the same file?
- Cause a hydration mismatch on purpose: render `new Date().toString()` in a server component, then again in a client child. Fix it.
- Walk through the path: user navigates to `/status-pages/[id]/status-reports`. List, in order, what runs on the server vs the client, when, and what data is fetched at each step.
- Take a client component in this repo that doesn't *need* to be client. Convert it. Verify the dashboard still works. Revert.
