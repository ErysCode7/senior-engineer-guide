# SEO Best Practices for Modern Web Applications

## Overview

Search Engine Optimization (SEO) is crucial for visibility and organic traffic. Modern web applications, especially SPAs, require special attention to SEO. This tutorial covers technical SEO implementation with Next.js and React.

## Understanding SEO Fundamentals

### What Search Engines Need

```typescript
// types/seo.ts
export interface SEORequirements {
  // Technical Requirements
  crawlable: boolean; // Can bots access your pages?
  indexable: boolean; // Should pages be indexed?
  fastLoading: boolean; // Core Web Vitals matter
  mobileResponsive: boolean; // Mobile-first indexing

  // Content Requirements
  uniqueContent: boolean; // No duplicate content
  structuredData: boolean; // Schema.org markup
  semanticHTML: boolean; // Proper HTML structure

  // Metadata
  titleTag: string; // Unique, descriptive titles
  metaDescription: string; // Compelling descriptions
  openGraph: object; // Social media previews
  canonicalURL: string; // Prevent duplicates
}
```

## Practical Use Cases

- **E-commerce**: Product pages ranking for shopping queries
- **Blogs**: Articles appearing in search results and news feeds
- **SaaS websites**: Landing pages ranking for industry keywords
- **Local businesses**: Appearing in local search and maps

## Step-by-Step Implementation

### 1. Meta Tags and Head Management

```typescript
// components/SEO.tsx
import Head from "next/head";

interface SEOProps {
  title: string;
  description: string;
  canonical?: string;
  ogImage?: string;
  ogType?: "website" | "article" | "product";
  twitterCard?: "summary" | "summary_large_image";
  noindex?: boolean;
  jsonLd?: object;
}

export function SEO({
  title,
  description,
  canonical,
  ogImage = "/og-default.jpg",
  ogType = "website",
  twitterCard = "summary_large_image",
  noindex = false,
  jsonLd,
}: SEOProps) {
  const siteUrl = process.env.NEXT_PUBLIC_SITE_URL;
  const fullTitle = `${title} | YourBrand`;
  const fullCanonical = canonical || siteUrl;
  const fullOgImage = ogImage.startsWith("http")
    ? ogImage
    : `${siteUrl}${ogImage}`;

  return (
    <Head>
      {/* Basic Meta Tags */}
      <title>{fullTitle}</title>
      <meta name="description" content={description} />
      <link rel="canonical" href={fullCanonical} />

      {/* Robots */}
      {noindex && <meta name="robots" content="noindex,nofollow" />}

      {/* Open Graph */}
      <meta property="og:type" content={ogType} />
      <meta property="og:title" content={fullTitle} />
      <meta property="og:description" content={description} />
      <meta property="og:url" content={fullCanonical} />
      <meta property="og:image" content={fullOgImage} />
      <meta property="og:image:width" content="1200" />
      <meta property="og:image:height" content="630" />
      <meta property="og:site_name" content="YourBrand" />

      {/* Twitter Card */}
      <meta name="twitter:card" content={twitterCard} />
      <meta name="twitter:title" content={fullTitle} />
      <meta name="twitter:description" content={description} />
      <meta name="twitter:image" content={fullOgImage} />
      <meta name="twitter:site" content="@yourbrand" />

      {/* JSON-LD Structured Data */}
      {jsonLd && (
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
        />
      )}
    </Head>
  );
}
```

### 2. Structured Data (Schema.org)

```typescript
// lib/structured-data.ts

export interface Article {
  title: string;
  description: string;
  image: string;
  datePublished: string;
  dateModified: string;
  author: {
    name: string;
    url: string;
  };
}

export function getArticleSchema(article: Article, url: string) {
  return {
    "@context": "https://schema.org",
    "@type": "Article",
    headline: article.title,
    description: article.description,
    image: article.image,
    datePublished: article.datePublished,
    dateModified: article.dateModified,
    author: {
      "@type": "Person",
      name: article.author.name,
      url: article.author.url,
    },
    publisher: {
      "@type": "Organization",
      name: "YourBrand",
      logo: {
        "@type": "ImageObject",
        url: `${process.env.NEXT_PUBLIC_SITE_URL}/logo.png`,
      },
    },
    mainEntityOfPage: {
      "@type": "WebPage",
      "@id": url,
    },
  };
}

export interface Product {
  name: string;
  description: string;
  image: string;
  price: number;
  currency: string;
  availability: "InStock" | "OutOfStock" | "PreOrder";
  rating?: {
    value: number;
    count: number;
  };
  brand: string;
}

export function getProductSchema(product: Product, url: string) {
  const schema: any = {
    "@context": "https://schema.org",
    "@type": "Product",
    name: product.name,
    description: product.description,
    image: product.image,
    brand: {
      "@type": "Brand",
      name: product.brand,
    },
    offers: {
      "@type": "Offer",
      url: url,
      priceCurrency: product.currency,
      price: product.price,
      availability: `https://schema.org/${product.availability}`,
    },
  };

  if (product.rating) {
    schema.aggregateRating = {
      "@type": "AggregateRating",
      ratingValue: product.rating.value,
      reviewCount: product.rating.count,
    };
  }

  return schema;
}

export function getBreadcrumbSchema(items: { name: string; url: string }[]) {
  return {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: items.map((item, index) => ({
      "@type": "ListItem",
      position: index + 1,
      name: item.name,
      item: item.url,
    })),
  };
}

export function getOrganizationSchema() {
  return {
    "@context": "https://schema.org",
    "@type": "Organization",
    name: "YourBrand",
    url: process.env.NEXT_PUBLIC_SITE_URL,
    logo: `${process.env.NEXT_PUBLIC_SITE_URL}/logo.png`,
    sameAs: [
      "https://twitter.com/yourbrand",
      "https://facebook.com/yourbrand",
      "https://linkedin.com/company/yourbrand",
    ],
    contactPoint: {
      "@type": "ContactPoint",
      telephone: "+1-555-555-5555",
      contactType: "Customer Service",
    },
  };
}
```

### 3. Implementing SEO on Pages

```typescript
// pages/blog/[slug].tsx
import { SEO } from "@/components/SEO";
import { getArticleSchema, getBreadcrumbSchema } from "@/lib/structured-data";

interface BlogPostProps {
  post: {
    slug: string;
    title: string;
    excerpt: string;
    content: string;
    coverImage: string;
    publishedAt: string;
    updatedAt: string;
    author: {
      name: string;
      url: string;
    };
    tags: string[];
  };
}

export default function BlogPost({ post }: BlogPostProps) {
  const url = `${process.env.NEXT_PUBLIC_SITE_URL}/blog/${post.slug}`;

  const articleSchema = getArticleSchema(
    {
      title: post.title,
      description: post.excerpt,
      image: post.coverImage,
      datePublished: post.publishedAt,
      dateModified: post.updatedAt,
      author: post.author,
    },
    url
  );

  const breadcrumbSchema = getBreadcrumbSchema([
    { name: "Home", url: process.env.NEXT_PUBLIC_SITE_URL! },
    { name: "Blog", url: `${process.env.NEXT_PUBLIC_SITE_URL}/blog` },
    { name: post.title, url },
  ]);

  return (
    <>
      <SEO
        title={post.title}
        description={post.excerpt}
        canonical={url}
        ogImage={post.coverImage}
        ogType="article"
        jsonLd={[articleSchema, breadcrumbSchema]}
      />

      <article>
        <header>
          <h1>{post.title}</h1>
          <time dateTime={post.publishedAt}>
            {new Date(post.publishedAt).toLocaleDateString()}
          </time>
        </header>

        <div dangerouslySetInnerHTML={{ __html: post.content }} />

        <footer>
          <div className="tags">
            {post.tags.map((tag) => (
              <a key={tag} href={`/blog/tag/${tag}`}>
                #{tag}
              </a>
            ))}
          </div>
        </footer>
      </article>
    </>
  );
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await getPostBySlug(params!.slug as string);

  return {
    props: { post },
    revalidate: 3600,
  };
};
```

### 4. Dynamic Sitemap Generation

```typescript
// pages/sitemap.xml.ts
import { GetServerSideProps } from "next";

function generateSiteMap(posts: any[], pages: any[]) {
  return `<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
            xmlns:news="http://www.google.com/schemas/sitemap-news/0.9"
            xmlns:xhtml="http://www.w3.org/1999/xhtml"
            xmlns:image="http://www.google.com/schemas/sitemap-image/1.1"
            xmlns:video="http://www.google.com/schemas/sitemap-video/1.1">
      ${pages
        .map((page) => {
          return `
        <url>
          <loc>${process.env.NEXT_PUBLIC_SITE_URL}${page.slug}</loc>
          <lastmod>${page.updatedAt}</lastmod>
          <changefreq>${page.changefreq}</changefreq>
          <priority>${page.priority}</priority>
        </url>
      `;
        })
        .join("")}
      ${posts
        .map((post) => {
          return `
        <url>
          <loc>${process.env.NEXT_PUBLIC_SITE_URL}/blog/${post.slug}</loc>
          <lastmod>${post.updatedAt}</lastmod>
          <changefreq>weekly</changefreq>
          <priority>0.8</priority>
        </url>
      `;
        })
        .join("")}
    </urlset>
  `;
}

export const getServerSideProps: GetServerSideProps = async ({ res }) => {
  // Fetch all posts and pages
  const posts = await getAllPosts();
  const pages = [
    {
      slug: "/",
      updatedAt: new Date().toISOString(),
      changefreq: "daily",
      priority: 1.0,
    },
    {
      slug: "/about",
      updatedAt: "2024-01-01",
      changefreq: "monthly",
      priority: 0.7,
    },
    {
      slug: "/contact",
      updatedAt: "2024-01-01",
      changefreq: "monthly",
      priority: 0.6,
    },
    // Add more static pages
  ];

  const sitemap = generateSiteMap(posts, pages);

  res.setHeader("Content-Type", "text/xml");
  res.setHeader(
    "Cache-Control",
    "public, s-maxage=86400, stale-while-revalidate"
  );
  res.write(sitemap);
  res.end();

  return {
    props: {},
  };
};

export default function Sitemap() {
  // This page is only accessed via /sitemap.xml
  return null;
}
```

### 5. Robots.txt Configuration

```typescript
// pages/robots.txt.ts
import { GetServerSideProps } from "next";

export const getServerSideProps: GetServerSideProps = async ({ res }) => {
  const robotsTxt = `
User-agent: *
Allow: /
Disallow: /api/
Disallow: /admin/
Disallow: /dashboard/
Disallow: /*?*utm_*
Disallow: /search?

User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

Sitemap: ${process.env.NEXT_PUBLIC_SITE_URL}/sitemap.xml

Crawl-delay: 1
  `.trim();

  res.setHeader("Content-Type", "text/plain");
  res.setHeader("Cache-Control", "public, max-age=86400");
  res.write(robotsTxt);
  res.end();

  return {
    props: {},
  };
};

export default function Robots() {
  return null;
}
```

### 6. Image SEO

```typescript
// components/SEOImage.tsx
import Image from "next/image";

interface SEOImageProps {
  src: string;
  alt: string;
  title?: string;
  width: number;
  height: number;
  caption?: string;
  priority?: boolean;
}

export function SEOImage({
  src,
  alt,
  title,
  width,
  height,
  caption,
  priority = false,
}: SEOImageProps) {
  return (
    <figure itemScope itemType="https://schema.org/ImageObject">
      <Image
        src={src}
        alt={alt}
        title={title || alt}
        width={width}
        height={height}
        priority={priority}
        loading={priority ? undefined : "lazy"}
        itemProp="contentUrl"
      />
      {caption && <figcaption itemProp="caption">{caption}</figcaption>}
      <meta itemProp="description" content={alt} />
    </figure>
  );
}
```

### 7. Internal Linking Strategy

```typescript
// components/RelatedPosts.tsx
import Link from "next/link";

interface Post {
  slug: string;
  title: string;
  excerpt: string;
  category: string;
}

interface RelatedPostsProps {
  currentPostId: string;
  category: string;
}

export function RelatedPosts({ currentPostId, category }: RelatedPostsProps) {
  const [posts, setPosts] = useState<Post[]>([]);

  useEffect(() => {
    // Fetch related posts based on category
    fetch(`/api/posts/related?category=${category}&exclude=${currentPostId}`)
      .then((res) => res.json())
      .then(setPosts);
  }, [currentPostId, category]);

  return (
    <aside className="related-posts">
      <h2>Related Articles</h2>
      <nav>
        <ul>
          {posts.map((post) => (
            <li key={post.slug}>
              <Link href={`/blog/${post.slug}`}>
                <h3>{post.title}</h3>
                <p>{post.excerpt}</p>
              </Link>
            </li>
          ))}
        </ul>
      </nav>
    </aside>
  );
}
```

### 8. Pagination SEO

```typescript
// pages/blog/page/[page].tsx
import { SEO } from "@/components/SEO";

interface BlogPageProps {
  posts: Post[];
  currentPage: number;
  totalPages: number;
}

export default function BlogPage({
  posts,
  currentPage,
  totalPages,
}: BlogPageProps) {
  const baseUrl = `${process.env.NEXT_PUBLIC_SITE_URL}/blog`;
  const canonical =
    currentPage === 1 ? baseUrl : `${baseUrl}/page/${currentPage}`;

  return (
    <>
      <SEO
        title={currentPage === 1 ? "Blog" : `Blog - Page ${currentPage}`}
        description="Latest articles and insights"
        canonical={canonical}
      />

      <Head>
        {/* Pagination meta tags */}
        {currentPage > 1 && (
          <link
            rel="prev"
            href={
              currentPage === 2 ? baseUrl : `${baseUrl}/page/${currentPage - 1}`
            }
          />
        )}
        {currentPage < totalPages && (
          <link rel="next" href={`${baseUrl}/page/${currentPage + 1}`} />
        )}
      </Head>

      <div>
        <h1>Blog</h1>

        {/* Post list */}
        <div className="posts">
          {posts.map((post) => (
            <article key={post.slug}>
              <h2>
                <Link href={`/blog/${post.slug}`}>{post.title}</Link>
              </h2>
              <p>{post.excerpt}</p>
            </article>
          ))}
        </div>

        {/* Pagination */}
        <nav aria-label="Pagination">
          <ul className="pagination">
            {currentPage > 1 && (
              <li>
                <Link
                  href={
                    currentPage === 2
                      ? "/blog"
                      : `/blog/page/${currentPage - 1}`
                  }
                  rel="prev"
                >
                  Previous
                </Link>
              </li>
            )}

            {Array.from({ length: totalPages }, (_, i) => i + 1).map((page) => (
              <li key={page}>
                <Link
                  href={page === 1 ? "/blog" : `/blog/page/${page}`}
                  aria-current={page === currentPage ? "page" : undefined}
                >
                  {page}
                </Link>
              </li>
            ))}

            {currentPage < totalPages && (
              <li>
                <Link href={`/blog/page/${currentPage + 1}`} rel="next">
                  Next
                </Link>
              </li>
            )}
          </ul>
        </nav>
      </div>
    </>
  );
}
```

### 9. Mobile-First and Responsive SEO

```typescript
// app/layout.tsx
export const metadata = {
  viewport: {
    width: "device-width",
    initialScale: 1,
    maximumScale: 5,
  },
  themeColor: [
    { media: "(prefers-color-scheme: light)", color: "#ffffff" },
    { media: "(prefers-color-scheme: dark)", color: "#000000" },
  ],
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <head>
        {/* Mobile optimization */}
        <meta name="format-detection" content="telephone=no" />
        <meta name="mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-status-bar-style" content="default" />

        {/* PWA manifest */}
        <link rel="manifest" href="/manifest.json" />

        {/* Favicons */}
        <link rel="icon" href="/favicon.ico" sizes="any" />
        <link rel="icon" href="/icon.svg" type="image/svg+xml" />
        <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

### 10. Handling Redirects and 404s

```typescript
// next.config.js
module.exports = {
  async redirects() {
    return [
      {
        source: "/old-blog/:slug",
        destination: "/blog/:slug",
        permanent: true, // 301 redirect
      },
      {
        source: "/products/:id",
        has: [
          {
            type: "query",
            key: "ref",
            value: "(?<ref>.*)",
          },
        ],
        destination: "/products/:id?source=:ref",
        permanent: false, // 302 redirect
      },
    ];
  },

  async rewrites() {
    return [
      {
        source: "/api/:path*",
        destination: "https://api.example.com/:path*",
      },
    ];
  },
};
```

```typescript
// pages/404.tsx
import Link from "next/link";
import { SEO } from "@/components/SEO";

export default function NotFound() {
  const [suggestions, setSuggestions] = useState<string[]>([]);

  useEffect(() => {
    // Get URL-based suggestions
    const path = window.location.pathname;
    fetch(`/api/search/suggestions?path=${encodeURIComponent(path)}`)
      .then((res) => res.json())
      .then((data) => setSuggestions(data.suggestions));
  }, []);

  return (
    <>
      <SEO
        title="Page Not Found"
        description="The page you're looking for doesn't exist"
        noindex
      />

      <div className="not-found">
        <h1>404 - Page Not Found</h1>
        <p>The page you're looking for doesn't exist or has been moved.</p>

        {suggestions.length > 0 && (
          <div className="suggestions">
            <h2>Did you mean:</h2>
            <ul>
              {suggestions.map((url) => (
                <li key={url}>
                  <Link href={url}>{url}</Link>
                </li>
              ))}
            </ul>
          </div>
        )}

        <Link href="/">Go to Homepage</Link>
      </div>
    </>
  );
}
```

## Best Practices

### 1. Content Quality

```typescript
// lib/content-quality.ts

export interface ContentAnalysis {
  wordCount: number;
  headingStructure: string[];
  readabilityScore: number;
  keywordDensity: Record<string, number>;
  hasUniqueContent: boolean;
}

export function analyzeContent(content: string): ContentAnalysis {
  const wordCount = content.split(/\s+/).length;
  const headings = extractHeadings(content);

  return {
    wordCount,
    headingStructure: headings,
    readabilityScore: calculateReadability(content),
    keywordDensity: calculateKeywordDensity(content),
    hasUniqueContent: wordCount > 300, // Minimum threshold
  };
}

// Aim for:
// - 300+ words for blog posts
// - Clear heading hierarchy (H1 -> H2 -> H3)
// - Readability score suitable for audience
// - Natural keyword usage (1-2% density)
```

### 2. Technical SEO Checklist

```typescript
export const technicalSEOChecklist = {
  performance: {
    "✓ Core Web Vitals pass": true,
    "✓ Mobile page speed > 90": true,
    "✓ Desktop page speed > 95": true,
    "✓ Images optimized": true,
  },

  indexability: {
    "✓ Robots.txt configured": true,
    "✓ Sitemap.xml generated": true,
    "✓ No indexing errors in Search Console": true,
    "✓ HTTPS enabled": true,
  },

  structure: {
    "✓ Proper heading hierarchy": true,
    "✓ Semantic HTML": true,
    "✓ Clean URL structure": true,
    "✓ Breadcrumb navigation": true,
  },

  metadata: {
    "✓ Unique titles (50-60 chars)": true,
    "✓ Compelling descriptions (150-160 chars)": true,
    "✓ Open Graph tags": true,
    "✓ Structured data implemented": true,
  },

  mobile: {
    "✓ Mobile-responsive design": true,
    "✓ Viewport meta tag": true,
    "✓ Touch-friendly UI": true,
    "✓ No mobile-specific errors": true,
  },
};
```

### 3. Avoiding Common SEO Mistakes

```typescript
// ❌ Bad: Duplicate content
export const getServerSideProps = async () => {
  return {
    props: {
      title: "Product Name",
      description: "Product Name - Buy Product Name online", // Duplicate
    },
  };
};

// ✅ Good: Unique, descriptive content
export const getServerSideProps = async () => {
  return {
    props: {
      title: "Premium Wireless Headphones with Noise Cancellation",
      description:
        "Experience crystal-clear audio with our premium wireless headphones featuring active noise cancellation and 30-hour battery life.",
    },
  };
};
```

```typescript
// ❌ Bad: Client-side rendering without fallback
function ProductPage() {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    fetchProduct().then(setProduct);
  }, []);

  return product ? <div>{product.name}</div> : <div>Loading...</div>;
}

// ✅ Good: Server-side rendering for SEO
export const getServerSideProps = async ({ params }) => {
  const product = await fetchProduct(params.id);
  return { props: { product } };
};

function ProductPage({ product }) {
  return <div>{product.name}</div>;
}
```

### 4. Monitoring SEO Performance

```typescript
// lib/seo-monitoring.ts

export async function trackSEOMetrics() {
  // Track in Google Search Console API
  const searchConsoleData = await fetch(
    "https://searchconsole.googleapis.com/webmasters/v3/sites/...",
    {
      headers: {
        Authorization: `Bearer ${process.env.SEARCH_CONSOLE_TOKEN}`,
      },
    }
  );

  return {
    clicks: 0,
    impressions: 0,
    ctr: 0,
    position: 0,
  };
}

// Monitor these metrics:
// - Organic traffic trends
// - Keyword rankings
// - Click-through rates
// - Core Web Vitals
// - Crawl errors
// - Index coverage
```

## Key Takeaways

1. **Server-Side Rendering**: Use SSR or SSG for content that needs to be indexed
2. **Structured Data**: Implement Schema.org markup for rich snippets
3. **Mobile-First**: Optimize for mobile devices - Google uses mobile-first indexing
4. **Performance**: Fast sites rank better - prioritize Core Web Vitals
5. **Unique Content**: Every page should have unique, valuable content
6. **Semantic HTML**: Use proper HTML5 elements for better accessibility and SEO
7. **Monitor & Iterate**: Use Search Console and Analytics to track and improve

SEO is an ongoing process, not a one-time setup. Regularly audit your site and adapt to search engine algorithm updates.
