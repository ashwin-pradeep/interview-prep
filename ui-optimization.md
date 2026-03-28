# UI Performance Optimization

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Core Web Vitals & Metrics](#core-web-vitals--metrics)
2. [Rendering Performance](#rendering-performance)
3. [JavaScript Optimization](#javascript-optimization)
4. [CSS & Layout Performance](#css--layout-performance)
5. [Image & Media Optimization](#image--media-optimization)
6. [Code Splitting & Lazy Loading](#code-splitting--lazy-loading)
7. [Caching Strategies](#caching-strategies)
8. [Network Optimization](#network-optimization)
9. [React-Specific Optimization](#react-specific-optimization)
10. [Monitoring & Profiling](#monitoring--profiling)

---

## Core Web Vitals & Metrics

### What are Core Web Vitals?

Core Web Vitals are Google's key metrics for measuring user experience:

| Metric | Measures | Good | Needs Improvement | Poor |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Loading speed | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | Responsiveness | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | ≤ 0.1 | ≤ 0.25 | > 0.25 |

```
User Journey:
Loading (LCP) → Interacting (INP) → Visual Stability (CLS)
```

**INP replaced FID** (First Input Delay) in March 2024. INP measures the **worst** interaction, not just the first.

---

### How do you measure and improve LCP?

**LCP measures** the time until the largest visible content element (image, video, text block) is rendered.

**Common LCP elements:** Hero images, large text blocks, video posters.

```js
// Measure LCP with PerformanceObserver
new PerformanceObserver((entryList) => {
  const entries = entryList.getEntries();
  const lastEntry = entries[entries.length - 1];
  console.log("LCP:", lastEntry.startTime);
}).observe({ type: "largest-contentful-paint", buffered: true });
```

**Optimization strategies:**

| Strategy | Impact | Implementation |
|---|---|---|
| Preload critical images | High | `<link rel="preload" as="image">` |
| Optimize server response | High | CDN, caching, edge rendering |
| Remove render-blocking resources | High | Async/defer scripts, critical CSS |
| Optimize images | Medium | WebP/AVIF, responsive `srcset`, lazy load below fold |
| Preconnect to origins | Medium | `<link rel="preconnect">` |

```html
<!-- Preload hero image -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<!-- Preconnect to CDN -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- fetchpriority for LCP image -->
<img src="/hero.webp" fetchpriority="high" alt="Hero" />
```

---

### How do you measure and improve INP?

**INP measures** the delay between user interaction and the next visual update.

```js
// Measure INP
new PerformanceObserver((entryList) => {
  for (const entry of entryList.getEntries()) {
    if (entry.interactionId) {
      console.log("Interaction:", entry.name, "Duration:", entry.duration);
    }
  }
}).observe({ type: "event", buffered: true });
```

**Optimization strategies:**
- Break up long tasks with `requestIdleCallback` or `scheduler.yield()`
- Use `startTransition` for non-urgent React updates
- Debounce/throttle event handlers
- Move heavy computation to Web Workers
- Reduce main thread work during interactions

```js
// Break long tasks
async function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i]);
    if (i % 100 === 0) {
      // Yield to the main thread
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }
}
```

---

### How do you measure and reduce CLS?

**CLS measures** unexpected layout shifts during the page lifecycle.

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Images without dimensions | Always set `width`/`height` or `aspect-ratio` |
| Ads/embeds without reserved space | Use placeholder containers |
| Dynamic content injection | Reserve space with min-height |
| Web fonts causing FOUT | `font-display: optional` or preload fonts |
| Late-loading CSS | Inline critical CSS |

```html
<!-- Always specify dimensions -->
<img src="photo.jpg" width="800" height="600" alt="Photo" />

<!-- CSS aspect-ratio -->
<style>
  .video-container {
    aspect-ratio: 16 / 9;
    width: 100%;
  }
</style>

<!-- Reserve space for dynamic content -->
<div style="min-height: 250px;">
  <!-- Ad loads here -->
</div>
```

```css
/* Prevent font swap layout shift */
@font-face {
  font-family: "CustomFont";
  src: url("/fonts/custom.woff2") format("woff2");
  font-display: optional; /* or swap with preload */
}
```

---

### What is TTFB and how do you optimize it?

**TTFB (Time to First Byte)** measures server responsiveness — time from request to first byte of response.

| Strategy | Implementation |
|---|---|
| CDN | Serve from edge locations |
| Server-side caching | Redis, Varnish, in-memory cache |
| Database optimization | Indexes, query optimization, connection pooling |
| HTTP/2 or HTTP/3 | Multiplexing, header compression |
| SSG/ISR | Pre-render pages at build time |
| Early hints (103) | Send preload hints before full response |

```
Client → CDN Edge (cache hit) → Response   (fast TTFB)
Client → CDN Edge (cache miss) → Origin → Response   (slower)
```

---

## Rendering Performance

### What is the Critical Rendering Path?

The sequence of steps the browser takes to convert HTML, CSS, and JS into pixels.

```
HTML → DOM Tree
                  → Render Tree → Layout → Paint → Composite
CSS  → CSSOM
```

**Steps:**
1. **Parse HTML** → DOM tree
2. **Parse CSS** → CSSOM tree
3. **Combine** → Render tree (visible elements only)
4. **Layout** — calculate positions and sizes
5. **Paint** — fill in pixels (colors, borders, shadows)
6. **Composite** — layer composition on GPU

**Optimization:** Minimize render-blocking resources, inline critical CSS, defer non-critical JS.

---

### What is reflow vs repaint?

| Concept | What triggers it | Cost | Example |
|---|---|---|---|
| **Reflow (Layout)** | Geometry changes | Expensive | Changing width, adding elements, font size |
| **Repaint** | Visual-only changes | Moderate | Color, visibility, background |
| **Composite** | Transform/opacity only | Cheap | `transform`, `opacity` |

```js
// ❌ Causes multiple reflows (layout thrashing)
for (let i = 0; i < 100; i++) {
  element.style.width = element.offsetWidth + 10 + "px"; // read + write in loop
}

// ✅ Batch reads, then writes
const width = element.offsetWidth; // read once
for (let i = 0; i < 100; i++) {
  elements[i].style.width = width + 10 + "px"; // write
}
```

---

### What are compositor-only properties?

Properties that can be animated without triggering layout or paint:

```css
/* ✅ Compositor-only — GPU accelerated, 60fps */
.animate {
  transform: translateX(100px);
  opacity: 0.5;
}

/* ❌ Triggers layout + paint */
.animate-bad {
  left: 100px;    /* layout */
  width: 200px;   /* layout */
  background: red; /* paint */
}

/* Promote to own layer for smoother animations */
.will-animate {
  will-change: transform;
  /* or */
  transform: translateZ(0); /* force GPU layer */
}
```

**Rule:** Animate only `transform` and `opacity` for smooth 60fps animations.

---

### What is the `requestAnimationFrame` API?

`requestAnimationFrame` schedules code to run before the next paint, synced to the display refresh rate.

```js
// Smooth animation loop
function animate() {
  element.style.transform = `translateX(${position}px)`;
  position += 2;
  if (position < 300) {
    requestAnimationFrame(animate);
  }
}
requestAnimationFrame(animate);

// Throttle scroll handler to frame rate
let ticking = false;
window.addEventListener("scroll", () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      updateHeader(window.scrollY);
      ticking = false;
    });
    ticking = true;
  }
});
```

---

## JavaScript Optimization

### What is debouncing vs throttling?

| Technique | Behavior | Use Case |
|---|---|---|
| **Debounce** | Waits until user stops, then executes | Search input, resize |
| **Throttle** | Executes at most once per interval | Scroll, mousemove |

```js
// Debounce — executes after delay of inactivity
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle — executes at most once per interval
function throttle(fn, limit) {
  let inThrottle;
  return (...args) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// Usage
const handleSearch = debounce((query) => fetchResults(query), 300);
const handleScroll = throttle(() => updatePosition(), 100);
```

---

### How do you optimize long-running JavaScript tasks?

**Problem:** Tasks > 50ms block the main thread, causing jank.

```js
// 1. requestIdleCallback — run during idle periods
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    processTask(tasks.shift());
  }
});

// 2. setTimeout chunking
function processInChunks(items, chunkSize = 100) {
  let index = 0;
  function processChunk() {
    const end = Math.min(index + chunkSize, items.length);
    for (; index < end; index++) {
      processItem(items[index]);
    }
    if (index < items.length) {
      setTimeout(processChunk, 0); // yield to main thread
    }
  }
  processChunk();
}

// 3. Web Workers for CPU-intensive work
const worker = new Worker("worker.js");
worker.postMessage({ data: largeDataset });
worker.onmessage = (e) => {
  renderResults(e.data);
};
```

---

### What is tree shaking?

Tree shaking removes unused code (dead code elimination) from bundles.

```js
// math.js
export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }  // unused

// app.js
import { add, multiply } from "./math";
// tree shaking removes `divide` from the bundle
```

**Requirements:**
- ES modules (`import`/`export`) — not CommonJS
- Bundler with tree shaking (Webpack, Vite/Rollup, esbuild)
- `sideEffects: false` in package.json for libraries

```json
// package.json
{
  "sideEffects": false,
  // or specify files with side effects
  "sideEffects": ["./src/polyfills.js", "*.css"]
}
```

---

### How do you reduce bundle size?

| Technique | Tool/Method |
|---|---|
| Tree shaking | ES modules + bundler |
| Code splitting | Dynamic `import()` |
| Analyze bundle | `webpack-bundle-analyzer`, `vite-bundle-visualizer` |
| Replace large libs | date-fns vs moment, preact vs react |
| Externalize deps | CDN for React, lodash |
| Minification | Terser, esbuild |
| Compression | Brotli > gzip |
| Import specific modules | `import pick from "lodash/pick"` |

```js
// ❌ Imports entire lodash (~70KB)
import { pick } from "lodash";

// ✅ Imports only pick (~1KB)
import pick from "lodash/pick";

// ❌ Imports entire icons library
import { FaHome } from "react-icons/fa";

// ✅ Import specific icon
import FaHome from "react-icons/fa/FaHome";
```

---

## CSS & Layout Performance

### What is critical CSS?

Critical CSS is the minimum CSS needed to render above-the-fold content. Inline it to avoid render-blocking.

```html
<head>
  <!-- Inline critical CSS -->
  <style>
    body { margin: 0; font-family: sans-serif; }
    .header { background: #333; color: white; padding: 1rem; }
    .hero { height: 80vh; display: flex; align-items: center; }
  </style>

  <!-- Load rest asynchronously -->
  <link rel="preload" href="/styles.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="/styles.css" /></noscript>
</head>
```

**Tools:** `critical` (npm), Critters (Webpack plugin), PurgeCSS.

---

### How does CSS containment improve performance?

`contain` tells the browser that an element's subtree is independent, enabling rendering optimizations.

```css
/* Full containment — most optimization */
.card {
  contain: content; /* layout + paint + style */
}

/* Specific containment */
.sidebar {
  contain: layout; /* layout changes don't affect parent */
}

/* content-visibility — skip rendering off-screen content */
.long-list-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 200px; /* estimated size */
}
```

`content-visibility: auto` can massively improve rendering of long pages by skipping off-screen content.

---

### What is `content-visibility: auto`?

It defers rendering of off-screen elements until they scroll into view.

```css
/* Before: browser renders all 1000 items */
/* After: only visible items are rendered */
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 80px; /* height hint for scrollbar */
}
```

**Benefits:**
- Reduces initial render time dramatically for long lists
- Browser reclaims rendering work for off-screen elements
- Works with native scrolling (no JS library needed)

---

## Image & Media Optimization

### What image formats should you use?

| Format | Best For | Compression | Browser Support |
|---|---|---|---|
| **AVIF** | Photos, illustrations | Best | Chrome, Firefox, Safari 16.4+ |
| **WebP** | Photos, animations | Very good | All modern browsers |
| **SVG** | Icons, logos, illustrations | Vector (scalable) | All |
| **PNG** | Transparency, screenshots | Lossless | All |
| **JPEG** | Photos (no transparency) | Good | All |

```html
<!-- Progressive enhancement with picture -->
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" width="1200" height="600" loading="lazy" />
</picture>
```

---

### How do you implement responsive images?

```html
<!-- srcset with sizes -->
<img
  srcset="
    photo-400w.webp 400w,
    photo-800w.webp 800w,
    photo-1200w.webp 1200w
  "
  sizes="
    (max-width: 600px) 400px,
    (max-width: 1200px) 800px,
    1200px
  "
  src="photo-800w.webp"
  alt="Responsive photo"
  loading="lazy"
  decoding="async"
/>

<!-- Art direction with picture -->
<picture>
  <source media="(max-width: 600px)" srcset="mobile.webp" />
  <source media="(max-width: 1200px)" srcset="tablet.webp" />
  <img src="desktop.webp" alt="Art directed" />
</picture>
```

---

### What is lazy loading and how do you implement it?

```html
<!-- Native lazy loading -->
<img src="photo.webp" loading="lazy" alt="Photo" />

<!-- Intersection Observer (more control) -->
<img data-src="photo.webp" class="lazy" alt="Photo" />
```

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      observer.unobserve(img);
    }
  });
}, { rootMargin: "200px" }); // start loading 200px before viewport

document.querySelectorAll(".lazy").forEach((img) => observer.observe(img));
```

**Important:** Don't lazy-load above-the-fold (LCP) images — use `fetchpriority="high"` instead.

---

## Code Splitting & Lazy Loading

### What is code splitting?

Code splitting divides your bundle into smaller chunks that load on demand.

```jsx
// Route-based splitting (most impactful)
const Home = React.lazy(() => import("./pages/Home"));
const Dashboard = React.lazy(() => import("./pages/Dashboard"));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}

// Component-based splitting
const HeavyChart = React.lazy(() => import("./components/HeavyChart"));

function Analytics() {
  const [showChart, setShowChart] = useState(false);
  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

### What is preloading and prefetching?

| Strategy | When it loads | Use Case |
|---|---|---|
| **Preload** | Current page, high priority | Critical resources (fonts, hero image) |
| **Prefetch** | Idle time, low priority | Next page resources |
| **Preconnect** | Establishes connection early | Third-party origins (CDN, API) |
| **DNS-prefetch** | DNS lookup only | Third-party origins |

```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
<link rel="prefetch" href="/dashboard.js" />
<link rel="preconnect" href="https://api.example.com" />
<link rel="dns-prefetch" href="https://analytics.example.com" />
```

```jsx
// Preload component on hover
const Dashboard = React.lazy(() => import("./Dashboard"));
const preloadDashboard = () => import("./Dashboard");

<Link to="/dashboard" onMouseEnter={preloadDashboard}>
  Dashboard
</Link>
```

---

## Caching Strategies

### What are the main browser caching strategies?

| Strategy | Header | Use Case |
|---|---|---|
| **Cache-First** | `Cache-Control: max-age=31536000, immutable` | Versioned static assets |
| **Network-First** | `Cache-Control: no-cache` | API responses, HTML |
| **Stale-While-Revalidate** | `Cache-Control: max-age=60, stale-while-revalidate=3600` | Semi-static content |
| **No Store** | `Cache-Control: no-store` | Sensitive data |

```
# Static assets with hash (immutable)
/assets/app.a1b2c3.js → Cache-Control: max-age=31536000, immutable

# HTML (always revalidate)
/index.html → Cache-Control: no-cache

# API responses (stale-while-revalidate)
/api/products → Cache-Control: max-age=60, stale-while-revalidate=3600
```

---

### What is a Service Worker and how does it help?

A Service Worker is a script that runs in the background, intercepting network requests for offline support and caching.

```js
// sw.js — Cache-first strategy
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request).then((response) => {
        const clone = response.clone();
        caches.open("v1").then((cache) => cache.put(event.request, clone));
        return response;
      });
    })
  );
});

// Register service worker
if ("serviceWorker" in navigator) {
  navigator.serviceWorker.register("/sw.js");
}
```

**Workbox** (by Google) simplifies service worker patterns:
```js
import { precacheAndRoute } from "workbox-precaching";
import { registerRoute } from "workbox-routing";
import { StaleWhileRevalidate } from "workbox-strategies";

precacheAndRoute(self.__WB_MANIFEST);

registerRoute(
  ({ url }) => url.pathname.startsWith("/api/"),
  new StaleWhileRevalidate({ cacheName: "api-cache" })
);
```

---

## Network Optimization

### What is HTTP/2 and how does it improve performance?

| Feature | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Connections | 6 per domain | 1 multiplexed |
| Header compression | None | HPACK |
| Server push | No | Yes |
| Request prioritization | No | Yes |
| Binary protocol | No (text) | Yes |

**Impact on bundling:** With HTTP/2, many small files can be as fast as fewer large bundles because of multiplexing.

---

### How does compression improve performance?

| Algorithm | Compression Ratio | Speed | Browser Support |
|---|---|---|---|
| **Brotli** | Best (~20% smaller than gzip) | Slower to compress | All modern |
| **Gzip** | Good | Fast | All |

```nginx
# Nginx config
gzip on;
gzip_types text/css application/javascript application/json;

# Brotli (if module installed)
brotli on;
brotli_types text/css application/javascript application/json;
```

**Pre-compression** at build time is even better:
```js
// vite.config.js
import viteCompression from "vite-plugin-compression";
export default {
  plugins: [
    viteCompression({ algorithm: "brotliCompress" }),
  ],
};
```

---

## React-Specific Optimization

### How do you profile React performance?

**React DevTools Profiler:**
1. Open DevTools → Profiler tab
2. Click Record → interact with app → Stop
3. Analyze flame graph for slow components

```jsx
// React.Profiler API
<Profiler id="Sidebar" onRender={onRenderCallback}>
  <Sidebar />
</Profiler>

function onRenderCallback(id, phase, actualDuration) {
  console.log(`${id} ${phase}: ${actualDuration}ms`);
  // Send to analytics
}
```

**why-did-you-render** — detects unnecessary re-renders:
```js
import whyDidYouRender from "@welldone-software/why-did-you-render";
whyDidYouRender(React, { trackAllPureComponents: true });
```

---

### What is the React optimization checklist?

| Level | Technique | Tool |
|---|---|---|
| **Component** | `React.memo`, `useMemo`, `useCallback` | React DevTools |
| **State** | Colocate state, split contexts | Profiler |
| **Rendering** | Virtualization, code splitting | react-window, React.lazy |
| **Network** | React Query caching, prefetching | @tanstack/react-query |
| **Bundle** | Tree shaking, dynamic imports | webpack-bundle-analyzer |
| **Images** | next/image, responsive images | Lighthouse |

---

### How do you handle large lists in React?

```jsx
// 1. Virtualization (react-window)
import { FixedSizeList } from "react-window";

function VirtualList({ items }) {
  return (
    <FixedSizeList height={600} itemCount={items.length} itemSize={50}>
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}

// 2. Pagination
function PaginatedList() {
  const [page, setPage] = useState(1);
  const { data } = useQuery({
    queryKey: ["items", page],
    queryFn: () => fetchItems(page),
  });
  return <List items={data?.items} onPageChange={setPage} />;
}

// 3. Infinite scroll
function InfiniteList() {
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ["items"],
    queryFn: ({ pageParam = 1 }) => fetchItems(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  const observerRef = useRef();
  const lastItemRef = useCallback((node) => {
    if (observerRef.current) observerRef.current.disconnect();
    observerRef.current = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting && hasNextPage) fetchNextPage();
    });
    if (node) observerRef.current.observe(node);
  }, [fetchNextPage, hasNextPage]);

  return (
    <div>
      {data?.pages.flat().map((item, i, arr) => (
        <div key={item.id} ref={i === arr.length - 1 ? lastItemRef : null}>
          {item.name}
        </div>
      ))}
    </div>
  );
}
```

---

## Monitoring & Profiling

### How do you set up performance monitoring?

```js
// Web Vitals library
import { onCLS, onINP, onLCP, onFCP, onTTFB } from "web-vitals";

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
    page: window.location.pathname,
  });
  navigator.sendBeacon("/api/analytics", body);
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

**Monitoring tools:**

| Tool | Type | Best For |
|---|---|---|
| **Lighthouse** | Lab | CI/CD audits, development |
| **Chrome DevTools** | Lab | Profiling, debugging |
| **PageSpeed Insights** | Field + Lab | Real user data (CrUX) |
| **Web Vitals Extension** | Field | Quick metric checks |
| **Sentry** | Field | Error + performance monitoring |
| **Datadog RUM** | Field | Enterprise real user monitoring |

---

### What is a performance budget?

A performance budget sets limits on metrics to prevent regressions.

```json
// budget.json (Lighthouse CI)
[
  {
    "path": "/*",
    "timings": [
      { "metric": "interactive", "budget": 3000 },
      { "metric": "largest-contentful-paint", "budget": 2500 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 200 },
      { "resourceType": "total", "budget": 500 }
    ]
  }
]
```

```yaml
# Lighthouse CI in GitHub Actions
- name: Lighthouse CI
  run: |
    npm install -g @lhci/cli
    lhci autorun --assert.preset=lighthouse:recommended
```

---

### What is the RAIL performance model?

| Phase | Goal | Budget |
|---|---|---|
| **R**esponse | React to user input | < 100ms |
| **A**nimation | Smooth frame rate | < 16ms (60fps) |
| **I**dle | Maximize idle time | < 50ms tasks |
| **L**oad | Deliver content fast | < 1000ms (interactive) |

```
User Action → Response (100ms) → Animation (16ms/frame) → Idle (50ms chunks) → Load (1s)
```

---

[← Back to README](./README.md)
