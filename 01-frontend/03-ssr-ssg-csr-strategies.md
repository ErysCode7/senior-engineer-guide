# SSR, SSG, and CSR Rendering Strategies

## Overview

Modern web applications can be rendered in different ways, each with distinct advantages. Understanding when and how to use Server-Side Rendering (SSR), Static Site Generation (SSG), and Client-Side Rendering (CSR) is crucial for building performant, SEO-friendly applications.

## What Are These Rendering Strategies?

### Client-Side Rendering (CSR)

- **Definition**: The browser downloads a minimal HTML page, then JavaScript renders the content dynamically
- **Process**: Server → Minimal HTML → Browser downloads JS → JS renders content

### Server-Side Rendering (SSR)

- **Definition**: HTML is generated on the server for each request
- **Process**: Request → Server renders HTML → Browser receives fully rendered page → Hydration

### Static Site Generation (SSG)

- **Definition**: HTML is generated at build time and reused for each request
- **Process**: Build time → Generate all pages → Deploy static files → Serve pre-rendered pages

## Practical Use Cases

### When to Use CSR

- **Dashboard applications** with authenticated users
- **Admin panels** that don't need SEO
- **Real-time applications** with frequent data updates
- **Interactive tools** that rely heavily on user input

### When to Use SSR

- **E-commerce product pages** that need SEO and fresh data
- **News websites** with frequently updated content
- **User-generated content** platforms (forums, social media)
- **Personalized content** that varies per user

### When to Use SSG

- **Marketing websites** with static content
- **Documentation sites** that don't change often
- **Blog posts** and articles
- **Landing pages** optimized for speed and SEO

## Step-by-Step Implementation (Next.js)

### 1. Client-Side Rendering (CSR)

```typescript
// pages/dashboard.tsx
import { useState, useEffect } from "react";

interface DashboardData {
  sales: number;
  users: number;
  revenue: number;
}

export default function Dashboard() {
  const [data, setData] = useState<DashboardData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Fetch data on the client side
    fetch("/api/dashboard")
      .then((res) => res.json())
      .then((data) => {
        setData(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading dashboard...</div>;
  if (!data) return <div>No data available</div>;

  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      <div className="metrics">
        <Metric label="Sales" value={data.sales} />
        <Metric label="Users" value={data.users} />
        <Metric label="Revenue" value={`$${data.revenue}`} />
      </div>
    </div>
  );
}

function Metric({ label, value }: { label: string; value: string | number }) {
  return (
    <div className="metric-card">
      <h3>{label}</h3>
      <p>{value}</p>
    </div>
  );
}
```

### 2. Server-Side Rendering (SSR)

```typescript
// pages/products/[id].tsx
import { GetServerSideProps } from "next";

interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  stock: number;
  reviews: Review[];
}

interface Review {
  id: string;
  rating: number;
  comment: string;
  author: string;
}

interface ProductPageProps {
  product: Product;
  userLocation: string;
}

export default function ProductPage({
  product,
  userLocation,
}: ProductPageProps) {
  return (
    <div className="product-page">
      <h1>{product.name}</h1>
      <p className="price">${product.price}</p>
      <p className="description">{product.description}</p>

      <div className="stock-info">
        {product.stock > 0 ? (
          <span className="in-stock">
            {product.stock} in stock - Ships to {userLocation}
          </span>
        ) : (
          <span className="out-of-stock">Out of stock</span>
        )}
      </div>

      <div className="reviews">
        <h2>Customer Reviews</h2>
        {product.reviews.map((review) => (
          <ReviewCard key={review.id} review={review} />
        ))}
      </div>
    </div>
  );
}

// This function runs on EVERY request
export const getServerSideProps: GetServerSideProps = async (context) => {
  const { id } = context.params!;
  const { req } = context;

  // Fetch product data on the server
  const productRes = await fetch(`${process.env.API_URL}/products/${id}`);
  const product = await productRes.json();

  // Get user location from IP (example)
  const userLocation = req.headers["x-forwarded-for"] || "US";

  // Return props to the page component
  return {
    props: {
      product,
      userLocation,
    },
  };
};

function ReviewCard({ review }: { review: Review }) {
  return (
    <div className="review-card">
      <div className="rating">{"⭐".repeat(review.rating)}</div>
      <p>{review.comment}</p>
      <p className="author">- {review.author}</p>
    </div>
  );
}
```

### 3. Static Site Generation (SSG)

```typescript
// pages/blog/[slug].tsx
import { GetStaticProps, GetStaticPaths } from "next";

interface BlogPost {
  slug: string;
  title: string;
  content: string;
  author: string;
  publishedAt: string;
  tags: string[];
}

interface BlogPostPageProps {
  post: BlogPost;
}

export default function BlogPostPage({ post }: BlogPostPageProps) {
  return (
    <article className="blog-post">
      <header>
        <h1>{post.title}</h1>
        <div className="meta">
          <span>By {post.author}</span>
          <time>{new Date(post.publishedAt).toLocaleDateString()}</time>
        </div>
        <div className="tags">
          {post.tags.map((tag) => (
            <span key={tag} className="tag">
              {tag}
            </span>
          ))}
        </div>
      </header>

      <div
        className="content"
        dangerouslySetInnerHTML={{ __html: post.content }}
      />
    </article>
  );
}

// Generate all possible paths at build time
export const getStaticPaths: GetStaticPaths = async () => {
  // Fetch all blog posts
  const res = await fetch(`${process.env.API_URL}/posts`);
  const posts: BlogPost[] = await res.json();

  // Generate paths for all posts
  const paths = posts.map((post) => ({
    params: { slug: post.slug },
  }));

  return {
    paths,
    fallback: "blocking", // or false, or true
  };
};

// Generate page content at build time
export const getStaticProps: GetStaticProps = async (context) => {
  const { slug } = context.params!;

  // Fetch post data
  const res = await fetch(`${process.env.API_URL}/posts/${slug}`);
  const post = await res.json();

  return {
    props: {
      post,
    },
    revalidate: 3600, // Revalidate every hour (ISR)
  };
};
```

### 4. Incremental Static Regeneration (ISR)

```typescript
// pages/products/category/[category].tsx
import { GetStaticProps, GetStaticPaths } from "next";

interface Product {
  id: string;
  name: string;
  price: number;
  image: string;
}

interface CategoryPageProps {
  category: string;
  products: Product[];
  lastUpdated: string;
}

export default function CategoryPage({
  category,
  products,
  lastUpdated,
}: CategoryPageProps) {
  return (
    <div className="category-page">
      <h1>{category}</h1>
      <p className="last-updated">Last updated: {lastUpdated}</p>

      <div className="product-grid">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}

export const getStaticPaths: GetStaticPaths = async () => {
  const categories = ["electronics", "clothing", "books", "home"];

  const paths = categories.map((category) => ({
    params: { category },
  }));

  return {
    paths,
    fallback: "blocking",
  };
};

export const getStaticProps: GetStaticProps = async (context) => {
  const { category } = context.params!;

  const res = await fetch(
    `${process.env.API_URL}/products/category/${category}`
  );
  const products = await res.json();

  return {
    props: {
      category,
      products,
      lastUpdated: new Date().toISOString(),
    },
    // Revalidate every 60 seconds
    // Next.js will regenerate the page in the background
    revalidate: 60,
  };
};

function ProductCard({ product }: { product: Product }) {
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p className="price">${product.price}</p>
    </div>
  );
}
```

### 5. Hybrid Approach (Combining Strategies)

```typescript
// pages/profile/[username].tsx
import { GetStaticProps, GetStaticPaths } from "next";
import { useState, useEffect } from "react";

interface UserProfile {
  username: string;
  bio: string;
  avatar: string;
  joinedDate: string;
}

interface Activity {
  id: string;
  type: string;
  description: string;
  timestamp: string;
}

interface ProfilePageProps {
  profile: UserProfile; // Static data
}

export default function ProfilePage({ profile }: ProfilePageProps) {
  // Client-side fetch for real-time activity
  const [activities, setActivities] = useState<Activity[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${profile.username}/activity`)
      .then((res) => res.json())
      .then((data) => {
        setActivities(data);
        setLoading(false);
      });
  }, [profile.username]);

  return (
    <div className="profile-page">
      {/* Static content from SSG */}
      <header className="profile-header">
        <img src={profile.avatar} alt={profile.username} />
        <h1>{profile.username}</h1>
        <p>{profile.bio}</p>
        <p className="joined">Joined {profile.joinedDate}</p>
      </header>

      {/* Dynamic content from CSR */}
      <section className="activity-feed">
        <h2>Recent Activity</h2>
        {loading ? (
          <div>Loading activities...</div>
        ) : (
          <ul>
            {activities.map((activity) => (
              <li key={activity.id}>
                <span className="type">{activity.type}</span>
                <span className="description">{activity.description}</span>
                <time>{new Date(activity.timestamp).toLocaleDateString()}</time>
              </li>
            ))}
          </ul>
        )}
      </section>
    </div>
  );
}

// Pre-generate popular profiles at build time
export const getStaticPaths: GetStaticPaths = async () => {
  const res = await fetch(`${process.env.API_URL}/users/popular`);
  const users = await res.json();

  const paths = users.map((user: UserProfile) => ({
    params: { username: user.username },
  }));

  return {
    paths,
    fallback: "blocking", // Generate other profiles on-demand
  };
};

export const getStaticProps: GetStaticProps = async (context) => {
  const { username } = context.params!;

  const res = await fetch(`${process.env.API_URL}/users/${username}`);
  const profile = await res.json();

  return {
    props: {
      profile,
    },
    revalidate: 3600, // Revalidate every hour
  };
};
```

## Performance Comparison

```typescript
// utils/performance-metrics.ts

/**
 * Performance characteristics of each rendering strategy
 */
export const renderingStrategies = {
  CSR: {
    ttfb: "Very Fast", // Time to First Byte
    fcp: "Slow", // First Contentful Paint
    lcp: "Slow", // Largest Contentful Paint
    tti: "Slow", // Time to Interactive
    seo: "Poor", // SEO friendliness
    serverLoad: "Very Low", // Server resource usage
    buildTime: "Fast", // Build duration
    cdnCacheable: "Yes", // Can be cached by CDN
    freshness: "Always Fresh", // Content freshness
  },
  SSR: {
    ttfb: "Slow",
    fcp: "Fast",
    lcp: "Fast",
    tti: "Medium",
    seo: "Excellent",
    serverLoad: "High",
    buildTime: "Fast",
    cdnCacheable: "Limited",
    freshness: "Always Fresh",
  },
  SSG: {
    ttfb: "Very Fast",
    fcp: "Very Fast",
    lcp: "Very Fast",
    tti: "Fast",
    seo: "Excellent",
    serverLoad: "Very Low",
    buildTime: "Slow",
    cdnCacheable: "Yes",
    freshness: "Stale (until rebuild)",
  },
  ISR: {
    ttfb: "Very Fast",
    fcp: "Very Fast",
    lcp: "Very Fast",
    tti: "Fast",
    seo: "Excellent",
    serverLoad: "Low",
    buildTime: "Medium",
    cdnCacheable: "Yes",
    freshness: "Mostly Fresh",
  },
};
```

## Best Practices

### 1. Choose the Right Strategy for Each Page

```typescript
// Example routing strategy
const routingStrategy = {
  // Static pages - Use SSG
  "/": "SSG",
  "/about": "SSG",
  "/pricing": "SSG",

  // Blog - Use SSG with ISR
  "/blog/[slug]": "SSG + ISR (revalidate: 3600)",

  // E-commerce - Use SSR for product pages
  "/products/[id]": "SSR",

  // User dashboards - Use CSR
  "/dashboard": "CSR",
  "/admin": "CSR",

  // Hybrid - Profile pages
  "/users/[username]": "SSG + ISR + CSR (hybrid)",
};
```

### 2. Optimize Data Fetching

```typescript
// lib/api-client.ts
import { unstable_cache } from "next/cache";

// Cache API responses for SSG/ISR
export async function getCachedProducts(category: string) {
  return unstable_cache(
    async () => {
      const res = await fetch(
        `${process.env.API_URL}/products?category=${category}`
      );
      return res.json();
    },
    [`products-${category}`],
    {
      revalidate: 300, // 5 minutes
      tags: ["products", category],
    }
  )();
}

// Parallel data fetching for SSR
export async function getProductWithReviews(id: string) {
  const [product, reviews] = await Promise.all([
    fetch(`${process.env.API_URL}/products/${id}`).then((r) => r.json()),
    fetch(`${process.env.API_URL}/products/${id}/reviews`).then((r) =>
      r.json()
    ),
  ]);

  return { product, reviews };
}
```

### 3. Handle Loading States Properly

```typescript
// components/LoadingBoundary.tsx
import { Suspense, ReactNode } from "react";

interface LoadingBoundaryProps {
  fallback?: ReactNode;
  children: ReactNode;
}

export function LoadingBoundary({ fallback, children }: LoadingBoundaryProps) {
  return (
    <Suspense fallback={fallback || <DefaultLoader />}>{children}</Suspense>
  );
}

function DefaultLoader() {
  return (
    <div className="loader">
      <div className="spinner" />
      <p>Loading...</p>
    </div>
  );
}

// Usage with CSR
function DashboardPage() {
  return (
    <LoadingBoundary fallback={<DashboardSkeleton />}>
      <DashboardContent />
    </LoadingBoundary>
  );
}
```

### 4. Implement Fallback Strategies

```typescript
// pages/docs/[...slug].tsx
export const getStaticPaths: GetStaticPaths = async () => {
  // Only pre-generate the most popular docs at build time
  const popularDocs = await getPopularDocs();

  const paths = popularDocs.map((doc) => ({
    params: { slug: doc.slug.split("/") },
  }));

  return {
    paths,
    // 'blocking' - SSR on first request, then cached
    // 'true' - Show fallback UI, then replace with real content
    // false - 404 for non-pre-generated paths
    fallback: "blocking",
  };
};
```

### 5. Use Streaming for Large SSR Pages

```typescript
// app/feed/page.tsx (Next.js 13+ App Router)
import { Suspense } from "react";

export default function FeedPage() {
  return (
    <div>
      <h1>Your Feed</h1>

      {/* Stream different sections independently */}
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations />
      </Suspense>

      <Suspense fallback={<TrendingSkeleton />}>
        <Trending />
      </Suspense>
    </div>
  );
}

async function Posts() {
  const posts = await fetchPosts();
  return <PostList posts={posts} />;
}
```

## Common Pitfalls to Avoid

### ❌ Don't Use SSR for Everything

```typescript
// Bad: Using SSR for a simple static about page
export const getServerSideProps = async () => {
  return { props: { data: "About us" } };
};
```

### ✅ Use SSG Instead

```typescript
// Good: Use SSG for static content
export const getStaticProps = async () => {
  return { props: { data: "About us" } };
};
```

### ❌ Don't Fetch on Both Server and Client

```typescript
// Bad: Double fetching
export const getServerSideProps = async () => {
  const data = await fetchData();
  return { props: { data } };
};

function Page({ data }) {
  useEffect(() => {
    // Don't fetch again!
    fetchData().then(setData);
  }, []);
}
```

### ✅ Use Server Data Directly

```typescript
// Good: Use the server-fetched data
export const getServerSideProps = async () => {
  const data = await fetchData();
  return { props: { data } };
};

function Page({ data }) {
  // Just use the data prop
  return <div>{data}</div>;
}
```

## Decision Flow Chart

```typescript
/**
 * Quick decision guide for choosing rendering strategy
 */
export function chooseRenderingStrategy(requirements: {
  needsSEO: boolean;
  dataChangeFrequency: "never" | "hourly" | "daily" | "realtime";
  personalized: boolean;
  buildsizeAcceptable: boolean;
}): string {
  const { needsSEO, dataChangeFrequency, personalized, buildsizeAcceptable } =
    requirements;

  if (personalized && !needsSEO) {
    return "CSR - Client-Side Rendering";
  }

  if (!needsSEO && dataChangeFrequency === "realtime") {
    return "CSR - Client-Side Rendering";
  }

  if (dataChangeFrequency === "never" || dataChangeFrequency === "daily") {
    if (buildsizeAcceptable) {
      return "SSG - Static Site Generation";
    } else {
      return "SSG with fallback: blocking";
    }
  }

  if (dataChangeFrequency === "hourly") {
    return "ISR - Incremental Static Regeneration";
  }

  return "SSR - Server-Side Rendering";
}

// Example usage
const strategy = chooseRenderingStrategy({
  needsSEO: true,
  dataChangeFrequency: "hourly",
  personalized: false,
  buildsizeAcceptable: true,
});
console.log(strategy); // "ISR - Incremental Static Regeneration"
```

## Key Takeaways

1. **CSR**: Best for authenticated dashboards and real-time apps where SEO isn't critical
2. **SSR**: Ideal for personalized or frequently changing content that needs SEO
3. **SSG**: Perfect for static content that doesn't change often (blogs, marketing pages)
4. **ISR**: Sweet spot for content that changes periodically but needs to be fast and SEO-friendly
5. **Hybrid**: Combine strategies on a single page for optimal UX (static shell + dynamic content)

Choose based on your specific requirements, not trends. The best architecture uses multiple strategies where appropriate.
