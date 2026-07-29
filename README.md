# Sports Live Scores Platform — Engineering Case Study

A production **Next.js 14 / TypeScript** platform serving live sports scores,
standings, and event data. It integrates four external live-data APIs behind a
resilient multi-tier cache, and is engineered for high read-traffic during live
matches while staying fast, type-safe, and search-indexable.

> This is a write-up of a live commercial product. The source is private; this
> repository documents the architecture and the engineering decisions behind it.

<img width="1920" height="1536" alt="screencapture-localhost-3000-2026-07-29-20_05_00" src="https://github.com/user-attachments/assets/95c73ad0-d515-4641-8754-0e76f1d674f8" />

<img width="1920" height="1920" alt="screencapture-localhost-3000-2026-07-29-20_05_00 (1)" src="https://github.com/user-attachments/assets/1507fb96-97d4-423d-83fc-9c0ed30390ce" />
<!-- ![Live scores](docs/scores.png) -->
<img width="1920" height="2064" alt="image" src="https://github.com/user-attachments/assets/d56d4f2d-46b6-4474-b6a9-52ecb7ef932a" />
<img width="1920" height="3490" alt="image" src="https://github.com/user-attachments/assets/2d972ada-1743-4844-bee0-ddea7d2080de" />

<!-- ![League standings](docs/standings.png) -->

---

## Highlights

- **Next.js 14 App Router** with a deliberate mix of SSG, ISR, and SSR per page type.
- **Four external sports APIs** unified behind one validated, cached data layer.
- **Resilient multi-tier caching** (distributed Redis + in-memory fallback) to survive
  upstream rate limits and traffic spikes during live events.
- **Runtime type safety** — every third-party payload is validated and normalized with
  Zod before it reaches the UI.
- **Performance & SEO engineered in**, not bolted on (Lighthouse scores below).
- **Tested** with Vitest (unit) and Playwright (end-to-end).

---

## Tech Stack

**Language & Framework:** TypeScript 5, Next.js 14 (App Router), React 18
**Styling / UI:** Tailwind CSS, Radix UI primitives, Framer Motion, Lucide, `cva`, `clsx`
**Data & Validation:** Zod, React Hook Form, Date-fns, Recharts
**Caching & Limits:** Upstash Redis, in-memory TTL fallback cache, Upstash rate limiter
**Testing:** Vitest, `@vitest/coverage-v8`, Playwright
**Infra & Perf:** Vercel (Serverless / Edge), Sharp image pipeline, Web Vitals monitoring, IndexNow automation

---

## Architecture

### Rendering strategy (chosen per page type)
| Strategy | Used for | Why |
|---|---|---|
| **SSG** | FAQ, Terms, static guides | Content never changes — serve pure static HTML |
| **ISR** | League overviews, team profiles, articles | Mostly stable; revalidate on an interval (1h–24h) |
| **SSR** | Live scores, in-progress matches | Data must be current to the second |

### Data flow

```
External Sports APIs
        │
        ▼
Rate limiter + circuit breaker
        │
        ▼
Multi-tier cache  (Redis → in-memory fallback)
        │
        ▼
Zod validation + response normalizer
        │
        ▼
Next.js App Router (Server Components / Route Handlers)
        │
        ▼
Client UI (real-time hydration)
```

### Caching strategy
Live sports traffic is spiky and upstream APIs enforce hard rate limits, so caching
is the core of the design:

- **Tier 1 — Distributed Redis:** short TTLs for volatile data (live scores, ~15–30s),
  longer TTLs for stable data (standings/rosters, up to 24h).
- **Tier 2 — In-memory fallback:** if Redis is slow or connection-limited, the client
  fails over to a local cache so responses stay fast instead of erroring.
- **Stale-while-revalidate:** serve the cached value immediately, refresh in the background.

**Measured impact (from production logs):**
- Redundant external API calls reduced by **~88%** during peak live-event traffic.
- Cached API-route latency of **~35ms** (down from ~450ms uncached).
- **>85%** cache hit rate during live-event spikes.

### Route handler pattern
Every API route in `app/api/[feature]/route.ts` follows one shape: parse & Zod-validate
input → rate-limit check → delegate to a service adapter in `lib/api/` → return a
standardized `{ data }` / `{ error, code }` JSON contract.

### Structure
```
app/
├── api/        # standardized route handlers (scores, news, events, search)
├── events/     # dynamic match & event views
├── leagues/    # standings & team hubs
└── (marketing)/# public content & indexation routes
components/
├── ui/         # Radix + Tailwind atomic components
├── homepage/   # dashboard features
└── shared/     # headers, footers, modals
lib/
├── api/        # API clients, circuit breakers, normalizers
├── cache/      # Redis + in-memory adapters
└── schema.ts   # Zod contracts
```

---

## Engineering decisions worth calling out

### 1. Resilient multi-tier cache
**Problem:** upstream APIs cap requests; live-event traffic would blow past the limit and
slow every page. **Approach:** one cache client wrapping Redis with an in-memory fallback,
volatility-based TTLs, and stale-while-revalidate. **Result:** ~88% fewer external API calls
at peak, cached-route latency down from ~450ms to ~35ms, and a >85% hit rate during live spikes.

### 2. Performance & Core Web Vitals
**Problem:** unoptimized images and web-font loading caused layout shift and slow LCP on
mobile. **Approach:** Next.js `<Image>` + Sharp compression, self-hosted Geist fonts to kill
font-driven layout shift, and component-level lazy loading. **Result:** eliminated layout
shift (CLS 0.00) and cut First Contentful Paint below ~1.1s.

### 3. Type-safe third-party data (Zod)
**Problem:** external APIs drift — missing or mistyped fields crashed client components at
runtime. **Approach:** strict Zod schemas parse and transform every payload into guaranteed
domain models before render. **Result:** eliminated a whole class of `undefined` runtime
errors from upstream payload drift.

### 4. Real progressive-download speed test
**Problem:** a prior speed-test feature used synthetic timers and reported inaccurate numbers
— a user-trust problem. **Approach:** a streaming route handler that pushes real byte chunks
and measures actual client throughput over time. **Result:** honest, dependency-free
bandwidth measurement in the browser.

### 5. Automated SEO / indexation
**Problem:** deep dynamic routes were slow to index. **Approach:** IndexNow pings wired into
the build, plus JSON-LD (`SportsEvent`, `BreadcrumbList`) generators. **Result:** new pages
submitted to search engines automatically; schema validates cleanly in Search Console.

---

## Measured results (Lighthouse)

| Category | Score |
|---|---|
| SEO | **100** |
| Best Practices | **100** |
| Accessibility | **96** |
| Performance | **94+ mobile / 98+ desktop** |

**Scale:** ~40+ routes · ~50+ custom components · ~18,000 lines of TypeScript/JS ·
4 external APIs · Vitest + Playwright test coverage.

---

## Selected code (sanitized, dummy data)

### Resilient cache wrapper
```typescript
interface CacheAdapter {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlSeconds: number): Promise<void>;
}

export class ResilientCacheService {
  constructor(
    private primaryCache: CacheAdapter,
    private fallbackCache: CacheAdapter
  ) {}

  async getOrFetch<T>(
    key: string,
    fetchFn: () => Promise<T>,
    ttlSeconds: number
  ): Promise<T> {
    try {
      const cached = await this.primaryCache.get<T>(key);
      if (cached !== null) return cached;
    } catch (err) {
      console.warn(`[Cache] Primary failed for "${key}", falling back:`, err);
    }

    try {
      const fallbackCached = await this.fallbackCache.get<T>(key);
      if (fallbackCached !== null) return fallbackCached;
    } catch {
      /* ignore fallback read error, fetch fresh */
    }

    const freshData = await fetchFn();
    this.primaryCache.set(key, freshData, ttlSeconds).catch(() => {});
    this.fallbackCache.set(key, freshData, ttlSeconds).catch(() => {});
    return freshData;
  }
}
```

### Zod normalizer for untrusted API payloads
```typescript
import { z } from "zod";

export const RawMatchPayloadSchema = z.object({
  event_id: z.string().or(z.number()).transform((v) => String(v)),
  home_team_name: z.string().default("Home Team"),
  away_team_name: z.string().default("Away Team"),
  status: z.enum(["SCHEDULED", "LIVE", "FINISHED"]).default("SCHEDULED"),
  scores: z
    .object({
      home: z.number().nullable().default(0),
      away: z.number().nullable().default(0),
    })
    .optional(),
});

export type MatchDomainModel = z.infer<typeof RawMatchPayloadSchema>;

export function normalizeMatchData(rawInput: unknown): MatchDomainModel {
  const result = RawMatchPayloadSchema.safeParse(rawInput);
  if (!result.success) {
    throw new Error("Data normalization failed: invalid payload structure");
  }
  return result.data;
}
```

### Streaming speed-test route handler
```typescript
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const sizeMb = Math.min(Math.max(Number(searchParams.get("size")) || 5, 1), 20);
  const totalBytes = sizeMb * 1024 * 1024;
  const chunkSize = 64 * 1024;

  const stream = new ReadableStream({
    start(controller) {
      let bytesSent = 0;
      const buffer = new Uint8Array(chunkSize).fill(0xaa);
      function push() {
        if (bytesSent >= totalBytes) return controller.close();
        controller.enqueue(buffer);
        bytesSent += chunkSize;
        setTimeout(push, 2);
      }
      push();
    },
  });

  return new NextResponse(stream, {
    headers: {
      "Content-Type": "application/octet-stream",
      "Content-Length": totalBytes.toString(),
      "Cache-Control": "no-store",
    },
  });
}
```
