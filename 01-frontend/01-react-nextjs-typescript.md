# React & Next.js with TypeScript: A Senior Engineer's Guide

## 📋 Overview

Modern React development has evolved significantly with Next.js 14+ introducing the App Router, Server Components, and enhanced TypeScript support. This tutorial covers production-ready patterns used in enterprise applications.

---

## 🎯 Use Cases

- **E-commerce platforms** - Product catalogs, shopping carts, checkout flows
- **SaaS dashboards** - Admin panels, analytics, user management
- **Content platforms** - Blogs, documentation sites, media applications
- **Real-time applications** - Chat apps, collaborative tools, live dashboards

---

## 🏗️ Project Setup

### 1. Create a New Next.js Project

```bash
npx create-next-app@latest my-app \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*"

cd my-app
```

### 2. Install Essential Dependencies

```bash
npm install zod @tanstack/react-query axios date-fns
npm install -D @types/node
```

### 3. Project Structure (Production-Ready)

```
src/
├── app/                    # Next.js App Router
│   ├── (auth)/            # Route groups
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── api/               # API routes
│   │   └── users/
│   ├── layout.tsx
│   └── page.tsx
├── components/            # React components
│   ├── ui/               # Reusable UI components
│   ├── features/         # Feature-specific components
│   └── layouts/          # Layout components
├── lib/                  # Utilities and configurations
│   ├── api/             # API client
│   ├── hooks/           # Custom hooks
│   ├── utils/           # Helper functions
│   └── validations/     # Zod schemas
├── types/               # TypeScript types
└── constants/           # Constants and enums
```

---

## 💻 Core Concepts with Code Examples

### 1. TypeScript Configuration

**`tsconfig.json`** (Production settings)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

### 2. Type-Safe Component Patterns

#### A. Generic Reusable Components

**`src/components/ui/Button.tsx`**

```typescript
import { ButtonHTMLAttributes, forwardRef } from "react";
import { cva, type VariantProps } from "class-variance-authority";

// CVA for type-safe variant styling
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        primary: "bg-blue-600 text-white hover:bg-blue-700",
        secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300",
        destructive: "bg-red-600 text-white hover:bg-red-700",
        ghost: "hover:bg-gray-100",
      },
      size: {
        sm: "h-9 px-3 text-sm",
        md: "h-10 px-4",
        lg: "h-11 px-8 text-lg",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "md",
    },
  }
);

export interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  isLoading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  (
    { className, variant, size, isLoading, children, disabled, ...props },
    ref
  ) => {
    return (
      <button
        ref={ref}
        className={buttonVariants({ variant, size, className })}
        disabled={disabled || isLoading}
        {...props}
      >
        {isLoading ? (
          <>
            <svg
              className="mr-2 h-4 w-4 animate-spin"
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
            >
              <circle
                className="opacity-25"
                cx="12"
                cy="12"
                r="10"
                stroke="currentColor"
                strokeWidth="4"
              />
              <path
                className="opacity-75"
                fill="currentColor"
                d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
              />
            </svg>
            Loading...
          </>
        ) : (
          children
        )}
      </button>
    );
  }
);

Button.displayName = "Button";
```

#### B. Type-Safe Form with Validation

**`src/lib/validations/user.ts`**

```typescript
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters").max(50),
  email: z.string().email("Invalid email address"),
  password: z
    .string()
    .min(8, "Password must be at least 8 characters")
    .regex(/[A-Z]/, "Password must contain at least one uppercase letter")
    .regex(/[a-z]/, "Password must contain at least one lowercase letter")
    .regex(/[0-9]/, "Password must contain at least one number"),
  role: z.enum(["user", "admin"]).default("user"),
  acceptedTerms: z.boolean().refine((val) => val === true, {
    message: "You must accept the terms and conditions",
  }),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
```

**`src/components/features/UserForm.tsx`**

```typescript
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { createUserSchema, type CreateUserInput } from "@/lib/validations/user";
import { Button } from "@/components/ui/Button";

export function UserForm() {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const {
    register,
    handleSubmit,
    formState: { errors },
    reset,
  } = useForm<CreateUserInput>({
    resolver: zodResolver(createUserSchema),
  });

  const onSubmit = async (data: CreateUserInput) => {
    setIsSubmitting(true);
    try {
      const response = await fetch("/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data),
      });

      if (!response.ok) throw new Error("Failed to create user");

      reset();
      alert("User created successfully!");
    } catch (error) {
      console.error(error);
      alert("Failed to create user");
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form
      onSubmit={handleSubmit(onSubmit)}
      className="space-y-4 max-w-md mx-auto"
    >
      <div>
        <label htmlFor="name" className="block text-sm font-medium mb-1">
          Name
        </label>
        <input
          id="name"
          type="text"
          {...register("name")}
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        {errors.name && (
          <p className="text-red-600 text-sm mt-1">{errors.name.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium mb-1">
          Email
        </label>
        <input
          id="email"
          type="email"
          {...register("email")}
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        {errors.email && (
          <p className="text-red-600 text-sm mt-1">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="password" className="block text-sm font-medium mb-1">
          Password
        </label>
        <input
          id="password"
          type="password"
          {...register("password")}
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        {errors.password && (
          <p className="text-red-600 text-sm mt-1">{errors.password.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="role" className="block text-sm font-medium mb-1">
          Role
        </label>
        <select
          id="role"
          {...register("role")}
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
        >
          <option value="user">User</option>
          <option value="admin">Admin</option>
        </select>
      </div>

      <div className="flex items-center">
        <input
          id="acceptedTerms"
          type="checkbox"
          {...register("acceptedTerms")}
          className="mr-2"
        />
        <label htmlFor="acceptedTerms" className="text-sm">
          I accept the terms and conditions
        </label>
      </div>
      {errors.acceptedTerms && (
        <p className="text-red-600 text-sm">{errors.acceptedTerms.message}</p>
      )}

      <Button type="submit" isLoading={isSubmitting} className="w-full">
        Create User
      </Button>
    </form>
  );
}
```

---

### 3. Server Components vs Client Components

#### Server Component (Default - Better Performance)

**`src/app/users/page.tsx`**

```typescript
import { User } from "@/types/user";

// This runs on the server, no JavaScript sent to client
async function getUsers(): Promise<User[]> {
  const res = await fetch("https://api.example.com/users", {
    cache: "no-store", // Always fresh data
    // OR: next: { revalidate: 60 } // Revalidate every 60 seconds
  });

  if (!res.ok) throw new Error("Failed to fetch users");
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">Users</h1>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {users.map((user) => (
          <div key={user.id} className="border rounded-lg p-4">
            <h2 className="text-xl font-semibold">{user.name}</h2>
            <p className="text-gray-600">{user.email}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

#### Client Component (Interactive)

**`src/components/features/UserCard.tsx`**

```typescript
"use client";

import { useState } from "react";
import { User } from "@/types/user";
import { Button } from "@/components/ui/Button";

interface UserCardProps {
  user: User;
}

export function UserCard({ user }: UserCardProps) {
  const [isFollowing, setIsFollowing] = useState(false);
  const [isLoading, setIsLoading] = useState(false);

  const handleFollow = async () => {
    setIsLoading(true);
    try {
      await fetch(`/api/users/${user.id}/follow`, { method: "POST" });
      setIsFollowing((prev) => !prev);
    } catch (error) {
      console.error("Failed to follow user:", error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="border rounded-lg p-4">
      <h2 className="text-xl font-semibold">{user.name}</h2>
      <p className="text-gray-600">{user.email}</p>
      <Button
        variant={isFollowing ? "secondary" : "primary"}
        size="sm"
        onClick={handleFollow}
        isLoading={isLoading}
        className="mt-2"
      >
        {isFollowing ? "Unfollow" : "Follow"}
      </Button>
    </div>
  );
}
```

---

### 4. Custom Hooks for Reusability

**`src/lib/hooks/useDebounce.ts`**

```typescript
import { useEffect, useState } from "react";

export function useDebounce<T>(value: T, delay: number = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

**`src/lib/hooks/useLocalStorage.ts`**

```typescript
import { useState, useEffect } from "react";

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((val: T) => T)) => void] {
  // Get from local storage or use initial value
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === "undefined") {
      return initialValue;
    }
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  // Return a wrapped version of useState's setter function that persists the new value to localStorage
  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      if (typeof window !== "undefined") {
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      }
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  };

  return [storedValue, setValue];
}
```

**Usage Example:**

```typescript
"use client";

import { useDebounce } from "@/lib/hooks/useDebounce";
import { useLocalStorage } from "@/lib/hooks/useLocalStorage";
import { useState, useEffect } from "react";

export function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState("");
  const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
  const debouncedSearch = useDebounce(searchTerm, 300);

  useEffect(() => {
    // Only fires when user stops typing for 300ms
    if (debouncedSearch) {
      console.log("Searching for:", debouncedSearch);
      // Perform API call here
    }
  }, [debouncedSearch]);

  return (
    <div>
      <input
        type="text"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
        className="border px-3 py-2 rounded"
      />
      <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
        Toggle Theme (Current: {theme})
      </button>
    </div>
  );
}
```

---

### 5. Error Handling & Loading States

**`src/app/products/[id]/page.tsx`**

```typescript
import { notFound } from "next/navigation";
import { Product } from "@/types/product";

async function getProduct(id: string): Promise<Product> {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    next: { revalidate: 3600 }, // Revalidate every hour
  });

  if (res.status === 404) {
    notFound(); // Renders not-found.tsx
  }

  if (!res.ok) {
    throw new Error("Failed to fetch product");
  }

  return res.json();
}

export default async function ProductPage({
  params,
}: {
  params: { id: string };
}) {
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-3xl font-bold">{product.name}</h1>
      <p className="text-gray-600 mt-2">{product.description}</p>
      <p className="text-2xl font-bold mt-4">${product.price}</p>
    </div>
  );
}
```

**`src/app/products/[id]/loading.tsx`**

```typescript
export default function Loading() {
  return (
    <div className="container mx-auto p-4">
      <div className="animate-pulse">
        <div className="h-8 bg-gray-200 rounded w-1/3 mb-4"></div>
        <div className="h-4 bg-gray-200 rounded w-full mb-2"></div>
        <div className="h-4 bg-gray-200 rounded w-2/3"></div>
      </div>
    </div>
  );
}
```

**`src/app/products/[id]/error.tsx`**

```typescript
"use client";

import { useEffect } from "react";
import { Button } from "@/components/ui/Button";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error("Product page error:", error);
  }, [error]);

  return (
    <div className="container mx-auto p-4 text-center">
      <h2 className="text-2xl font-bold text-red-600 mb-4">
        Something went wrong!
      </h2>
      <p className="text-gray-600 mb-4">{error.message}</p>
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

**`src/app/products/[id]/not-found.tsx`**

```typescript
import Link from "next/link";
import { Button } from "@/components/ui/Button";

export default function NotFound() {
  return (
    <div className="container mx-auto p-4 text-center">
      <h2 className="text-2xl font-bold mb-4">Product Not Found</h2>
      <p className="text-gray-600 mb-4">
        Could not find the requested product.
      </p>
      <Link href="/products">
        <Button>Back to Products</Button>
      </Link>
    </div>
  );
}
```

---

## 🎨 Best Practices

### 1. Component Organization

```typescript
// ✅ GOOD: Separate concerns
export function ProductCard({ product }: { product: Product }) {
  return (
    <article className="border rounded-lg p-4">
      <ProductImage src={product.image} alt={product.name} />
      <ProductDetails product={product} />
      <ProductActions product={product} />
    </article>
  );
}

// ❌ BAD: Everything in one component
export function ProductCard({ product }: { product: Product }) {
  // 200 lines of mixed logic...
}
```

### 2. Use TypeScript Strictly

```typescript
// ✅ GOOD: Explicit types
interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}

function getUserName(user: User): string {
  return user.name;
}

// ❌ BAD: Any types
function getUserName(user: any): any {
  return user.name;
}
```

### 3. Optimize Re-renders

```typescript
// ✅ GOOD: Memoize expensive calculations
const MemoizedList = memo(function ProductList({
  products,
}: {
  products: Product[];
}) {
  const sortedProducts = useMemo(
    () => products.sort((a, b) => b.price - a.price),
    [products]
  );

  return (
    <div>
      {sortedProducts.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
});

// ✅ GOOD: useCallback for event handlers passed as props
function ParentComponent() {
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []);

  return <ChildComponent onClick={handleClick} />;
}
```

### 4. Environment Variables

**`.env.local`**

```env
# Public (accessible in browser)
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_APP_NAME=My App

# Private (server-side only)
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
API_SECRET_KEY=your-secret-key
```

**`src/lib/env.ts`**

```typescript
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_API_URL: z.string().url(),
  NEXT_PUBLIC_APP_NAME: z.string(),
  DATABASE_URL: z.string(),
  API_SECRET_KEY: z.string().min(32),
});

export const env = envSchema.parse({
  NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  DATABASE_URL: process.env.DATABASE_URL,
  API_SECRET_KEY: process.env.API_SECRET_KEY,
});
```

### 5. Code Splitting & Dynamic Imports

```typescript
import dynamic from "next/dynamic";

// Heavy component loaded only when needed
const DynamicChart = dynamic(() => import("@/components/Chart"), {
  loading: () => <p>Loading chart...</p>,
  ssr: false, // Don't render on server if it uses browser APIs
});

export function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && <DynamicChart data={chartData} />}
    </div>
  );
}
```

---

## 🚀 Performance Tips

1. **Use Server Components by default** - Only use 'use client' when needed
2. **Implement proper caching** - Use `revalidate` strategically
3. **Optimize images** - Use Next.js `<Image>` component
4. **Code split large components** - Use dynamic imports
5. **Minimize client-side JavaScript** - Move logic to server when possible

---

## 🔍 Common Pitfalls

| ❌ Mistake                         | ✅ Solution                             |
| ---------------------------------- | --------------------------------------- |
| Using `'use client'` everywhere    | Only add when you need interactivity    |
| Not handling loading states        | Use `loading.tsx` and Suspense          |
| Fetching on client unnecessarily   | Use Server Components for data fetching |
| Ignoring TypeScript errors         | Fix all errors before deployment        |
| Not memoizing expensive operations | Use `useMemo` and `useCallback`         |

---

## 📚 Next Steps

- [State Management & API Integration →](./02-state-management-api.md)
- [Rendering Strategies →](./03-rendering-strategies.md)

---

**🎯 Key Takeaway:** Modern React with Next.js and TypeScript provides type safety, excellent performance, and developer experience. Always prefer Server Components, use TypeScript strictly, and optimize for user experience.
