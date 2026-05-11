# Web Rendering Strategies: CSR, SSR, SSG & ISR

A comprehensive guide to understanding Client-Side Rendering, Server-Side Rendering, Static Site Generation, and Incremental Static Regeneration — with setup examples and a decision framework. Click ⭐ if you like the project. Pull Requests are highly appreciated.

---

## Table of Contents

- [Overview](#overview)
- [1. Client-Side Rendering (CSR)](#1-client-side-rendering-csr)
- [2. Server-Side Rendering (SSR)](#2-server-side-rendering-ssr)
- [3. Static Site Generation (SSG)](#3-static-site-generation-ssg)
- [4. Incremental Static Regeneration (ISR)](#4-incremental-static-regeneration-isr)
- [Decision Guide](#decision-guide)
- [Quick Comparison Table](#quick-comparison-table)

---

## Overview

Modern web applications have four primary rendering strategies, each with distinct trade-offs across performance, SEO, and content freshness. Choosing the right one depends on your content type, update frequency, and user experience requirements.

| Strategy | Where it Renders | When it Renders |
|---|---|---|
| CSR | Browser (Client) | On every user visit |
| SSR | Server | On every request |
| SSG | Build server | At build time |
| ISR | Build server + CDN | At build time + background revalidation |

---

## 1. Client-Side Rendering (CSR)

The server sends a minimal HTML shell. The browser downloads JavaScript and renders the full UI on the client side.

```
Browser Request → Server sends empty HTML + JS bundle → Browser executes JS → Page renders
```

### How It Works

1. Server responds with a near-empty HTML file and a `<script>` tag
2. Browser downloads and parses JavaScript
3. React (or another framework) mounts and fetches data via API
4. UI renders in the browser

### Basic Setup (React + Vite)

```bash
npm create vite@latest my-csr-app -- --template react
cd my-csr-app
npm install
npm run dev
```

```jsx
// src/App.jsx
import { useState, useEffect } from 'react'

export default function App() {
  const [posts, setPosts] = useState([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Data is fetched in the browser AFTER initial render
    fetch('https://jsonplaceholder.typicode.com/posts?_limit=5')
      .then(res => res.json())
      .then(data => {
        setPosts(data)
        setLoading(false)
      })
  }, [])

  if (loading) return <p>Loading...</p>

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### What the Initial HTML Looks Like

```html
<!-- index.html — what the server actually sends -->
<!DOCTYPE html>
<html>
  <body>
    <div id="root"></div>          <!-- empty! content comes later -->
    <script src="/assets/index.js"></script>
  </body>
</html>
```

### Characteristics

- **Build Time:** Near-instant (no data processing at build)
- **Rendering Time:** Slow initial paint; fast subsequent navigations
- **SEO:** Poor by default (crawlers may not execute JS)
- **Dynamic Content:** Excellent (everything is fetched at runtime)
- **Content Updates:** Instant (no rebuild needed)

---

## 2. Server-Side Rendering (SSR)

For every incoming request, the server fetches data and renders a fully populated HTML page, which is then sent to the browser.

```
Browser Request → Server fetches data → Server renders HTML → Browser receives full page
```

### How It Works

1. User requests a page
2. Server runs your component/template code
3. Server fetches required data
4. Server returns complete, data-populated HTML
5. Browser displays content immediately; JS hydrates for interactivity

### Basic Setup (Next.js)

```bash
npx create-next-app@latest my-ssr-app
cd my-ssr-app
npm run dev
```

```jsx
// app/posts/page.jsx  (Next.js App Router — SSR by default)

async function getPosts() {
  // This runs on the SERVER for every request
  const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=5', {
    cache: 'no-store'   // <-- disables caching; forces SSR behaviour
  })
  return res.json()
}

export default async function PostsPage() {
  const posts = await getPosts()

  return (
    <main>
      <h1>Latest Posts</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  )
}
```

```jsx
// pages/posts.jsx  (Next.js Pages Router equivalent)
export async function getServerSideProps(context) {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=5')
  const posts = await res.json()

  return {
    props: { posts }   // passed to the component as props
  }
}

export default function PostsPage({ posts }) {
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

### What the Server Sends

```html
<!-- Fully rendered HTML — crawlers and users see content immediately -->
<ul>
  <li>Post title one</li>
  <li>Post title two</li>
  <li>Post title three</li>
</ul>
```

### Characteristics

- **Build Time:** Fast (no pre-rendering)
- **Rendering Time:** Server latency on every request; TTFB can be high under load
- **SEO:** Excellent (full HTML is always available)
- **Dynamic Content:** Excellent (fresh data on every request)
- **Content Updates:** Instant (no rebuild needed)

---

## 3. Static Site Generation (SSG)

All pages are fully rendered at **build time** and deployed as static HTML files to a CDN.

```
Build time → Server fetches data → Renders all pages to static HTML → Deploys to CDN
Browser Request → CDN serves pre-built HTML (no server needed)
```

### How It Works

1. At build time, the framework fetches all required data
2. It generates a complete HTML file for every page
3. These files are deployed to a CDN
4. Users receive HTML instantly from the CDN edge — no server computation

### Basic Setup (Next.js)

```bash
npx create-next-app@latest my-ssg-app
cd my-ssg-app
npm run build    # generates all static pages
npm run start
```

```jsx
// app/blog/[slug]/page.jsx  (Next.js App Router)

// Tell Next.js which paths to pre-build
export async function generateStaticParams() {
  const posts = await fetch('https://jsonplaceholder.typicode.com/posts').then(r => r.json())

  return posts.map(post => ({
    slug: String(post.id)
  }))
}

// Fetch data for each page at build time
async function getPost(slug) {
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${slug}`, {
    cache: 'force-cache'   // <-- SSG: cache permanently
  })
  return res.json()
}

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  )
}
```

```jsx
// pages/blog/[slug].jsx  (Next.js Pages Router equivalent)
export async function getStaticPaths() {
  const posts = await fetch('https://jsonplaceholder.typicode.com/posts').then(r => r.json())

  return {
    paths: posts.map(post => ({ params: { slug: String(post.id) } })),
    fallback: false   // 404 for unknown paths
  }
}

export async function getStaticProps({ params }) {
  const post = await fetch(`https://jsonplaceholder.typicode.com/posts/${params.slug}`)
    .then(r => r.json())

  return { props: { post } }
}

export default function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  )
}
```

### Characteristics

- **Build Time:** Slow (scales with number of pages)
- **Rendering Time:** Fastest possible (CDN edge, no server)
- **SEO:** Excellent
- **Dynamic Content:** None (content is frozen at build time)
- **Content Updates:** Requires a full rebuild and redeploy

---

## 4. Incremental Static Regeneration (ISR)

ISR combines SSG's performance with SSR's freshness. Pages are statically generated but automatically **revalidated in the background** after a defined interval — without a full rebuild.

```
Build time → Generates static HTML → Deploys to CDN
First request after interval → Server regenerates page in background
Next request → Serves newly regenerated page
```

### How It Works

1. Pages are pre-built like SSG at initial deploy
2. A `revalidate` duration is set per page (e.g., 60 seconds)
3. After the interval expires, the next request triggers a background rebuild
4. Subsequent visitors get the fresh page; no user waits for revalidation

### Basic Setup (Next.js)

```jsx
// app/products/page.jsx  (Next.js App Router)

async function getProducts() {
  const res = await fetch('https://fakestoreapi.com/products', {
    next: { revalidate: 60 }   // <-- revalidate every 60 seconds
  })
  return res.json()
}

export default async function ProductsPage() {
  const products = await getProducts()

  return (
    <main>
      <h1>Products</h1>
      <ul>
        {products.map(product => (
          <li key={product.id}>
            {product.title} — ${product.price}
          </li>
        ))}
      </ul>
    </main>
  )
}
```

```jsx
// pages/products.jsx  (Next.js Pages Router equivalent)
export async function getStaticProps() {
  const products = await fetch('https://fakestoreapi.com/products').then(r => r.json())

  return {
    props: { products },
    revalidate: 60   // <-- regenerate at most once every 60 seconds
  }
}

export default function ProductsPage({ products }) {
  return (
    <ul>
      {products.map(p => (
        <li key={p.id}>{p.title} — ${p.price}</li>
      ))}
    </ul>
  )
}
```

### On-Demand ISR (Next.js 12.2+)

Trigger revalidation immediately via a webhook — useful after a CMS publish event.

```js
// pages/api/revalidate.js
export default async function handler(req, res) {
  // Validate a secret token first in production
  if (req.query.secret !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: 'Invalid token' })
  }

  try {
    await res.revalidate('/products')   // regenerate this path immediately
    return res.json({ revalidated: true })
  } catch (err) {
    return res.status(500).send('Error revalidating')
  }
}
```

### Characteristics

- **Build Time:** Fast (only a subset of pages built initially)
- **Rendering Time:** Fast (CDN-served; background revalidation is invisible to users)
- **SEO:** Excellent
- **Dynamic Content:** Good (data is fresh within the revalidation window)
- **Content Updates:** Semi-automatic (interval-based or on-demand via webhook)

---

## Decision Guide

Use this flowchart logic to pick the right strategy.

```
Start: What kind of content does this page serve?
│
├─ Always the same for every user, rarely changes
│   └─ Is build time acceptable even with 1000s of pages?
│       ├─ Yes → SSG ✅
│       └─ No  → ISR (with a long revalidate interval) ✅
│
├─ Same content for all users, but changes regularly (hours/days)
│   └─ ISR (with a short revalidate interval) ✅
│
├─ Unique per user OR changes by the second (live prices, feeds)
│   └─ Does SEO matter for this page?
│       ├─ Yes → SSR ✅
│       └─ No  → CSR ✅
│
└─ Behind authentication (dashboards, admin panels, user profiles)
    └─ CSR ✅ (SEO is irrelevant; data is user-specific)
```

### Factor-by-Factor Breakdown

#### Build Time

| Strategy | Impact | Notes |
|---|---|---|
| CSR | ⚡ Negligible | No data fetching at build; ships a JS bundle only |
| SSR | ⚡ Negligible | Pages are not pre-rendered; build is just compilation |
| SSG | 🐢 Scales with pages | 10,000 pages = long builds; use `fallback: true` for large sites |
| ISR | ⚡ Fast | Only critical pages built upfront; others generated on demand |

**Guidance:** If build time is a bottleneck, avoid pure SSG for large catalogs. Use ISR with `fallback: 'blocking'` to build pages on first visit and cache them afterwards.

---

#### Rendering Time (Time to First Byte / First Contentful Paint)

| Strategy | TTFB | FCP | Notes |
|---|---|---|---|
| CSR | ⚡ Fast | 🐢 Slow | HTML arrives fast but is empty; JS must run before content appears |
| SSR | 🐢 Slow | ⚡ Fast | Server does work before responding; content appears as soon as HTML arrives |
| SSG | ⚡ Fastest | ⚡ Fastest | Pre-built HTML served from CDN edge |
| ISR | ⚡ Fastest | ⚡ Fastest | Same as SSG; revalidation is invisible to the user |

**Guidance:** For the fastest perceived performance, prefer SSG or ISR. If you need live data and good SEO, SSR with caching headers is the next best option.

---

#### SEO

| Strategy | SEO Quality | Reason |
|---|---|---|
| CSR | ❌ Poor | Initial HTML is empty; Googlebot must execute JS to see content (unreliable) |
| SSR | ✅ Excellent | Full HTML on every request; crawlers always see complete content |
| SSG | ✅ Excellent | Pre-built full HTML; instant crawlability |
| ISR | ✅ Excellent | Behaves like SSG from a crawler's perspective |

**Guidance:** Any public-facing page that should rank in search engines needs SSR, SSG, or ISR. Use CSR only for pages behind authentication or where SEO is not a concern.

---

#### Dynamic Content

| Strategy | Freshness | Suitable For |
|---|---|---|
| CSR | ✅ Real-time | User dashboards, live feeds, anything behind a login |
| SSR | ✅ Real-time | Personalised pages, live pricing, social feeds with SEO needs |
| SSG | ❌ Frozen at build | Documentation, marketing pages, blog posts |
| ISR | ⚠️ Within revalidation window | Product catalogs, news articles, semi-live data |

**Guidance:** If data changes by the second and SEO matters, use SSR. If data changes hourly or daily, ISR with an appropriate interval is more efficient.

---

#### Content Updates

| Strategy | Update Mechanism | Latency to Users |
|---|---|---|
| CSR | API call at runtime | Instant |
| SSR | Server fetches on every request | Instant |
| SSG | Full rebuild + redeploy | Minutes (CI/CD pipeline) |
| ISR | Background revalidation or webhook | Seconds to minutes |

**Guidance:** For content managed via a CMS (e.g., Contentful, Sanity), ISR with on-demand revalidation triggered by publish webhooks gives you the speed of SSG with near-instant updates.

---

## Quick Comparison Table

| Factor | CSR | SSR | SSG | ISR |
|---|---|---|---|---|
| **Build Time** | ⚡ Instant | ⚡ Instant | 🐢 Long (scales with pages) | ⚡ Fast |
| **Rendering Time** | 🐢 Slow initial paint | ⚠️ Server latency | ⚡ Fastest | ⚡ Fastest |
| **SEO** | ❌ Poor | ✅ Excellent | ✅ Excellent | ✅ Excellent |
| **Dynamic Content** | ✅ Real-time | ✅ Real-time | ❌ Build-time only | ⚠️ Within interval |
| **Content Updates** | ✅ Instant | ✅ Instant | ❌ Rebuild required | ✅ Interval or webhook |
| **Infrastructure** | CDN / static host | Node.js server | CDN / static host | CDN + serverless |
| **Best For** | Dashboards, auth pages | Live SEO pages | Docs, marketing | Catalogs, news, blogs |

---

## Common Use-Case Recommendations

| Page Type | Recommended Strategy | Reasoning |
|---|---|---|
| Marketing / landing page | SSG | Rarely changes; maximum performance |
| Blog / documentation | SSG or ISR | Content is static; ISR if editors publish frequently |
| E-commerce product listing | ISR (60–300s) | Prices/stock change but not per-second |
| E-commerce product page | ISR + on-demand | Needs fast load + fresh stock/price after CMS update |
| News / article page | ISR (60s) | Fresh content without full rebuilds |
| User dashboard | CSR | Personalised, behind auth, SEO irrelevant |
| Search results page | SSR | Query-dependent, must be indexed, real-time |
| Admin panel | CSR | Auth-gated, no SEO requirement |
| Social feed with SEO | SSR | User-specific + needs to be indexable |

---


*Built with Next.js examples. The same concepts apply to Nuxt (Vue), SvelteKit, Remix, Astro, and other modern meta-frameworks — only the API surface differs.*