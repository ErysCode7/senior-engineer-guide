# Performance Optimization Techniques

## Overview

Performance optimization is critical for user experience, SEO rankings, and conversion rates. A 1-second delay can reduce conversions by 7%. This tutorial covers practical techniques to make your web applications blazingly fast.

## Core Web Vitals

### Understanding the Metrics

```typescript
// types/performance.ts
export interface WebVitals {
  // Largest Contentful Paint - Loading performance
  LCP: number; // Target: < 2.5s

  // First Input Delay - Interactivity
  FID: number; // Target: < 100ms

  // Cumulative Layout Shift - Visual stability
  CLS: number; // Target: < 0.1

  // First Contentful Paint
  FCP: number; // Target: < 1.8s

  // Time to Interactive
  TTI: number; // Target: < 3.8s
}
```

## Practical Use Cases

- **E-commerce sites**: Faster load times = higher conversion rates
- **Content platforms**: Better engagement with instant page loads
- **Mobile apps**: Reduced data usage and battery consumption
- **SEO**: Google prioritizes fast websites in search rankings

## Step-by-Step Implementation

### 1. Image Optimization

```typescript
// components/OptimizedImage.tsx
import Image from "next/image";
import { useState } from "react";

interface OptimizedImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false,
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true);

  return (
    <div className="relative overflow-hidden">
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        priority={priority} // For above-the-fold images
        loading={priority ? undefined : "lazy"}
        quality={85} // Balance between quality and size
        placeholder="blur"
        blurDataURL={generateBlurDataURL(width, height)}
        onLoadingComplete={() => setIsLoading(false)}
        className={`
          duration-300 ease-in-out
          ${isLoading ? "scale-110 blur-lg" : "scale-100 blur-0"}
        `}
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      />
    </div>
  );
}

// Generate a tiny blur placeholder
function generateBlurDataURL(w: number, h: number): string {
  const canvas = document.createElement("canvas");
  canvas.width = w;
  canvas.height = h;
  const ctx = canvas.getContext("2d");
  if (ctx) {
    ctx.fillStyle = "#f0f0f0";
    ctx.fillRect(0, 0, w, h);
  }
  return canvas.toDataURL();
}
```

### 2. Code Splitting and Lazy Loading

```typescript
// pages/dashboard.tsx
import dynamic from "next/dynamic";
import { Suspense, lazy } from "react";

// Dynamic imports for Next.js
const HeavyChart = dynamic(() => import("@/components/HeavyChart"), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Don't render on server if not needed
});

const Analytics = dynamic(() => import("@/components/Analytics"), {
  loading: () => <AnalyticsSkeleton />,
});

// React lazy loading
const Comments = lazy(() => import("@/components/Comments"));

export default function Dashboard() {
  const [showAnalytics, setShowAnalytics] = useState(false);

  return (
    <div className="dashboard">
      <h1>Dashboard</h1>

      {/* Load immediately - critical content */}
      <QuickStats />

      {/* Load when visible - use Intersection Observer */}
      <LazyLoad>
        <HeavyChart data={chartData} />
      </LazyLoad>

      {/* Load on user interaction */}
      {showAnalytics ? (
        <Analytics />
      ) : (
        <button onClick={() => setShowAnalytics(true)}>Show Analytics</button>
      )}

      {/* Load with Suspense */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId="123" />
      </Suspense>
    </div>
  );
}

// Intersection Observer for lazy loading
function LazyLoad({ children }: { children: React.ReactNode }) {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {isVisible ? children : <div style={{ minHeight: "400px" }} />}
    </div>
  );
}
```

### 3. Bundle Size Optimization

```typescript
// next.config.js
const withBundleAnalyzer = require("@next/bundle-analyzer")({
  enabled: process.env.ANALYZE === "true",
});

module.exports = withBundleAnalyzer({
  // Enable SWC minification (faster than Terser)
  swcMinify: true,

  // Optimize package imports
  modularizeImports: {
    lodash: {
      transform: "lodash/{{member}}",
    },
    "@mui/material": {
      transform: "@mui/material/{{member}}",
    },
  },

  // Experimental features for better performance
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ["lucide-react", "date-fns"],
  },

  // Webpack optimizations
  webpack: (config, { isServer }) => {
    if (!isServer) {
      // Don't include these packages in client bundle
      config.resolve.fallback = {
        fs: false,
        net: false,
        tls: false,
      };
    }

    // Tree shaking
    config.optimization.usedExports = true;

    return config;
  },
});
```

```typescript
// utils/optimized-imports.ts

// ❌ Bad: Imports entire library
import _ from "lodash";
import * as dateFns from "date-fns";

// ✅ Good: Import only what you need
import debounce from "lodash/debounce";
import formatDistance from "date-fns/formatDistance";

// ❌ Bad: Large icon library
import { FaUser, FaHome, FaSearch } from "react-icons/fa";

// ✅ Good: Use a smaller, tree-shakeable library
import { User, Home, Search } from "lucide-react";
```

### 4. Caching Strategies

```typescript
// lib/cache.ts
import { unstable_cache } from "next/cache";

// Server-side caching with Next.js
export const getCachedData = unstable_cache(
  async (key: string) => {
    const response = await fetch(`${process.env.API_URL}/data/${key}`);
    return response.json();
  },
  ["data-cache"],
  {
    revalidate: 3600, // 1 hour
    tags: ["data"],
  }
);

// Client-side caching with SWR
import useSWR from "swr";

export function useOptimizedData(endpoint: string) {
  return useSWR(endpoint, fetcher, {
    revalidateOnFocus: false,
    revalidateOnReconnect: false,
    dedupingInterval: 60000, // 1 minute
    // Keep previous data while fetching new
    keepPreviousData: true,
  });
}

// React Query caching
import { useQuery } from "@tanstack/react-query";

export function useProductQuery(id: string) {
  return useQuery({
    queryKey: ["product", id],
    queryFn: () => fetchProduct(id),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    // Enable for lists that don't change often
    refetchOnWindowFocus: false,
  });
}
```

### 5. Database Query Optimization

```typescript
// lib/db-optimization.ts
import { prisma } from "./prisma";

// ❌ Bad: N+1 query problem
async function getBadUserPosts(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
  });

  // Separate query for each post's author
  const posts = await prisma.post.findMany({
    where: { authorId: userId },
  });

  // Another query for each post's comments
  for (const post of posts) {
    post.comments = await prisma.comment.findMany({
      where: { postId: post.id },
    });
  }

  return posts;
}

// ✅ Good: Use include to fetch related data in one query
async function getOptimizedUserPosts(userId: string) {
  return prisma.post.findMany({
    where: { authorId: userId },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          avatar: true,
        },
      },
      comments: {
        include: {
          author: {
            select: {
              id: true,
              name: true,
            },
          },
        },
        orderBy: {
          createdAt: "desc",
        },
        take: 5, // Only latest 5 comments
      },
      _count: {
        select: {
          likes: true,
          comments: true,
        },
      },
    },
    orderBy: {
      createdAt: "desc",
    },
  });
}

// ✅ Even better: Use select to only fetch needed fields
async function getLeanUserPosts(userId: string) {
  return prisma.post.findMany({
    where: { authorId: userId },
    select: {
      id: true,
      title: true,
      createdAt: true,
      author: {
        select: {
          name: true,
          avatar: true,
        },
      },
      _count: {
        select: {
          likes: true,
          comments: true,
        },
      },
    },
  });
}
```

### 6. React Performance Optimizations

```typescript
// components/OptimizedList.tsx
import { memo, useMemo, useCallback } from "react";
import { FixedSizeList as VirtualList } from "react-window";

interface Item {
  id: string;
  name: string;
  value: number;
}

interface OptimizedListProps {
  items: Item[];
  onItemClick: (id: string) => void;
}

// Memoize the entire component
export const OptimizedList = memo(function OptimizedList({
  items,
  onItemClick,
}: OptimizedListProps) {
  // Memoize expensive calculations
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => b.value - a.value);
  }, [items]);

  // Memoize callbacks to prevent child re-renders
  const handleClick = useCallback(
    (id: string) => {
      onItemClick(id);
    },
    [onItemClick]
  );

  // Virtual scrolling for large lists
  return (
    <VirtualList
      height={600}
      itemCount={sortedItems.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <ListItem
          key={sortedItems[index].id}
          item={sortedItems[index]}
          onClick={handleClick}
          style={style}
        />
      )}
    </VirtualList>
  );
});

// Memoize list items
const ListItem = memo(function ListItem({
  item,
  onClick,
  style,
}: {
  item: Item;
  onClick: (id: string) => void;
  style: React.CSSProperties;
}) {
  return (
    <div style={style} onClick={() => onClick(item.id)}>
      <span>{item.name}</span>
      <span>{item.value}</span>
    </div>
  );
});
```

### 7. API Response Optimization

```typescript
// pages/api/products.ts
import { NextApiRequest, NextApiResponse } from "next";
import { z } from "zod";

// Compress responses
import compression from "compression";

// Pagination schema
const querySchema = z.object({
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().min(1).max(100).default(20),
  sort: z.enum(["price", "name", "date"]).default("date"),
});

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Set aggressive caching headers
  res.setHeader(
    "Cache-Control",
    "public, s-maxage=60, stale-while-revalidate=120"
  );

  // Parse and validate query params
  const query = querySchema.parse(req.query);

  // Use cursor-based pagination for better performance
  const products = await prisma.product.findMany({
    take: query.limit,
    skip: (query.page - 1) * query.limit,
    orderBy: {
      [query.sort]: "desc",
    },
    select: {
      id: true,
      name: true,
      price: true,
      image: true,
      // Don't send unnecessary data
      // description: false,
      // internalNotes: false,
    },
  });

  // Send minimal response
  return res.status(200).json({
    data: products,
    pagination: {
      page: query.page,
      limit: query.limit,
      hasMore: products.length === query.limit,
    },
  });
}
```

### 8. Font Loading Optimization

```typescript
// app/layout.tsx (Next.js 13+)
import { Inter, Roboto_Mono } from "next/font/google";

// Optimize font loading
const inter = Inter({
  subsets: ["latin"],
  display: "swap", // Use fallback font while loading
  variable: "--font-inter",
  preload: true,
  // Only load specific weights you need
  weight: ["400", "600", "700"],
});

const robotoMono = Roboto_Mono({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-roboto-mono",
  weight: ["400", "700"],
});

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

```css
/* styles/globals.css */

/* Use font-display: swap for custom fonts */
@font-face {
  font-family: "CustomFont";
  src: url("/fonts/custom-font.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

/* Optimize font loading with preload */
/* Add to <head> */
/* <link rel="preload" href="/fonts/custom-font.woff2" as="font" type="font/woff2" crossorigin> */
```

### 9. Prefetching and Preloading

```typescript
// components/OptimizedLink.tsx
import Link from "next/link";
import { useRouter } from "next/router";

interface OptimizedLinkProps {
  href: string;
  children: React.ReactNode;
  prefetch?: boolean;
}

export function OptimizedLink({
  href,
  children,
  prefetch = true,
}: OptimizedLinkProps) {
  const router = useRouter();

  // Prefetch on hover
  const handleMouseEnter = () => {
    if (prefetch) {
      router.prefetch(href);
    }
  };

  return (
    <Link
      href={href}
      onMouseEnter={handleMouseEnter}
      prefetch={false} // Disable automatic prefetch
    >
      {children}
    </Link>
  );
}
```

```typescript
// Preload critical resources
// pages/_document.tsx
import { Html, Head, Main, NextScript } from "next/document";

export default function Document() {
  return (
    <Html>
      <Head>
        {/* Preconnect to external domains */}
        <link rel="preconnect" href="https://api.example.com" />
        <link rel="dns-prefetch" href="https://cdn.example.com" />

        {/* Preload critical assets */}
        <link
          rel="preload"
          href="/fonts/inter-var.woff2"
          as="font"
          type="font/woff2"
          crossOrigin="anonymous"
        />

        {/* Preload critical images */}
        <link rel="preload" as="image" href="/hero-image.webp" />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  );
}
```

### 10. Performance Monitoring

```typescript
// lib/performance-monitoring.ts
import { onCLS, onFID, onLCP, onFCP, onTTFB } from "web-vitals";

interface Metric {
  name: string;
  value: number;
  rating: "good" | "needs-improvement" | "poor";
}

export function initPerformanceMonitoring() {
  // Report to analytics service
  function sendToAnalytics(metric: Metric) {
    // Send to your analytics service
    fetch("/api/analytics", {
      method: "POST",
      body: JSON.stringify(metric),
      headers: { "Content-Type": "application/json" },
      // Use keepalive to ensure the request completes even if user navigates away
      keepalive: true,
    });
  }

  // Monitor Core Web Vitals
  onCLS(sendToAnalytics);
  onFID(sendToAnalytics);
  onLCP(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}

// Custom performance marks
export function measureCustomPerformance(
  name: string,
  startMark: string,
  endMark: string
) {
  performance.mark(endMark);
  performance.measure(name, startMark, endMark);

  const measure = performance.getEntriesByName(name)[0];
  console.log(`${name}: ${measure.duration}ms`);

  return measure.duration;
}

// Usage
export function usePerformanceTracking(componentName: string) {
  useEffect(() => {
    const startMark = `${componentName}-start`;
    performance.mark(startMark);

    return () => {
      measureCustomPerformance(
        `${componentName}-render`,
        startMark,
        `${componentName}-end`
      );
    };
  }, [componentName]);
}
```

## Best Practices

### 1. Prioritize Above-the-Fold Content

```typescript
// components/Hero.tsx
export function Hero() {
  return (
    <section className="hero">
      {/* Use priority for above-the-fold images */}
      <Image
        src="/hero-image.jpg"
        alt="Hero"
        width={1200}
        height={600}
        priority // Preload this image
        quality={90}
      />

      {/* Inline critical CSS */}
      <style jsx>{`
        .hero {
          height: 600px;
          display: flex;
          align-items: center;
          justify-content: center;
        }
      `}</style>
    </section>
  );
}
```

### 2. Implement Progressive Enhancement

```typescript
// components/ProgressiveForm.tsx
export function ProgressiveForm() {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  return (
    <form>
      {/* Basic HTML form works without JS */}
      <input type="text" name="query" required />
      <button type="submit">Search</button>

      {/* Enhanced features load after mount */}
      {mounted && (
        <>
          <AutoComplete />
          <RecentSearches />
        </>
      )}
    </form>
  );
}
```

### 3. Use Resource Hints Wisely

```typescript
// Use rel="preconnect" for critical third-party origins
<link rel="preconnect" href="https://fonts.googleapis.com" />

// Use rel="dns-prefetch" for non-critical origins
<link rel="dns-prefetch" href="https://analytics.example.com" />

// Use rel="prefetch" for next page resources
<link rel="prefetch" href="/dashboard.js" />

// Use rel="preload" for critical current page resources
<link rel="preload" href="/critical-styles.css" as="style" />
```

### 4. Optimize Third-Party Scripts

```typescript
// components/ThirdPartyScripts.tsx
import Script from "next/script";

export function ThirdPartyScripts() {
  return (
    <>
      {/* Load analytics after page is interactive */}
      <Script
        src="https://analytics.example.com/script.js"
        strategy="lazyOnload"
      />

      {/* Load critical scripts early but don't block rendering */}
      <Script
        src="https://checkout.stripe.com/v3.js"
        strategy="afterInteractive"
      />

      {/* Load with custom logic */}
      <Script
        id="fb-pixel"
        strategy="afterInteractive"
        dangerouslySetInnerHTML={{
          __html: `
            !function(f,b,e,v,n,t,s)
            {if(f.fbq)return;n=f.fbq=function(){...}}
          `,
        }}
      />
    </>
  );
}
```

### 5. Implement Smart Retry Logic

```typescript
// lib/fetch-with-retry.ts
export async function fetchWithRetry<T>(
  url: string,
  options: RequestInit = {},
  maxRetries = 3
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, {
        ...options,
        // Add timeout
        signal: AbortSignal.timeout(5000),
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      lastError = error as Error;

      // Exponential backoff
      if (i < maxRetries - 1) {
        await new Promise((resolve) =>
          setTimeout(resolve, Math.pow(2, i) * 1000)
        );
      }
    }
  }

  throw lastError!;
}
```

## Performance Checklist

```typescript
export const performanceChecklist = {
  images: {
    "✓ Using next/image or optimized image component": true,
    "✓ Serving WebP/AVIF formats": true,
    "✓ Implementing lazy loading": true,
    "✓ Using appropriate sizes": true,
    "✓ Compressing images": true,
  },

  code: {
    "✓ Code splitting implemented": true,
    "✓ Tree shaking enabled": true,
    "✓ Unused dependencies removed": true,
    "✓ Bundle size under 200KB (gzipped)": true,
    "✓ Dynamic imports for heavy components": true,
  },

  caching: {
    "✓ HTTP caching headers set": true,
    "✓ CDN configured": true,
    "✓ Service worker for offline support": false,
    "✓ Client-side caching (SWR/React Query)": true,
  },

  rendering: {
    "✓ Using appropriate rendering strategy": true,
    "✓ Avoiding unnecessary re-renders": true,
    "✓ Memoizing expensive computations": true,
    "✓ Virtual scrolling for long lists": true,
  },

  fonts: {
    "✓ Using font-display: swap": true,
    "✓ Preloading critical fonts": true,
    "✓ Subsetting fonts": true,
    "✓ Using variable fonts": false,
  },
};
```

## Key Takeaways

1. **Measure First**: Use Lighthouse, PageSpeed Insights, and Web Vitals before optimizing
2. **Prioritize**: Focus on what impacts users most (LCP, FID, CLS)
3. **Images Matter**: They're usually the biggest bottleneck - optimize them first
4. **Code Split**: Don't send code users don't need immediately
5. **Cache Aggressively**: Use all layers - CDN, browser, server, and client-side caches
6. **Monitor Continuously**: Performance degrades over time - track it in production

Performance is not a one-time task, it's an ongoing commitment. Set performance budgets and enforce them in your CI/CD pipeline.
