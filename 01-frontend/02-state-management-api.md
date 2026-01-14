# State Management & API Integration

## 📋 Overview

Modern state management goes beyond Redux. This guide covers Zustand for lightweight state, React Query for server state, and best practices for API integration in production applications.

---

## 🎯 Use Cases

- **Global UI state** - Theme, modals, notifications, sidebar state
- **Server data caching** - User profiles, product catalogs, real-time data
- **Form state** - Multi-step forms, draft persistence
- **Authentication state** - User sessions, permissions, tokens

---

## 🏗️ State Management Architecture

```
┌─────────────────────────────────────┐
│         Application State           │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐  ┌─────────────┐ │
│  │ Client State │  │Server State │ │
│  │  (Zustand)   │  │(React Query)│ │
│  └──────────────┘  └─────────────┘ │
│                                     │
│  ┌──────────────┐  ┌─────────────┐ │
│  │  URL State   │  │ Form State  │ │
│  │ (Next.js)    │  │(React Hook  │ │
│  │              │  │   Form)     │ │
│  └──────────────┘  └─────────────┘ │
└─────────────────────────────────────┘
```

---

## 💻 Setup

```bash
npm install zustand @tanstack/react-query axios
npm install -D @tanstack/eslint-plugin-query
```

---

## 1️⃣ React Query for Server State

### Setup Query Provider

**`src/lib/providers/QueryProvider.tsx`**

```typescript
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            // Don't refetch on window focus in production
            refetchOnWindowFocus: false,
            // Retry failed requests
            retry: 1,
            // Cache for 5 minutes
            staleTime: 5 * 60 * 1000,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === "development" && <ReactQueryDevtools />}
    </QueryClientProvider>
  );
}
```

**`src/app/layout.tsx`**

```typescript
import { QueryProvider } from "@/lib/providers/QueryProvider";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

---

### API Client Setup

**`src/lib/api/client.ts`**

```typescript
import axios from "axios";

// Create axios instance with base configuration
export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || "https://api.example.com",
  timeout: 10000,
  headers: {
    "Content-Type": "application/json",
  },
});

// Request interceptor - add auth token
apiClient.interceptors.request.use(
  (config) => {
    // Get token from localStorage or cookies
    const token = localStorage.getItem("access_token");
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor - handle errors globally
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Handle 401 - refresh token
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem("refresh_token");
        const response = await axios.post("/api/auth/refresh", {
          refreshToken,
        });
        const { accessToken } = response.data;

        localStorage.setItem("access_token", accessToken);
        apiClient.defaults.headers.Authorization = `Bearer ${accessToken}`;
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;

        return apiClient(originalRequest);
      } catch (refreshError) {
        // Redirect to login
        window.location.href = "/login";
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

---

### Type-Safe API Functions

**`src/lib/api/users.ts`**

```typescript
import { apiClient } from "./client";
import { z } from "zod";

// Define schemas for runtime validation
export const userSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  avatar: z.string().url().nullable(),
  role: z.enum(["user", "admin"]),
  createdAt: z.string().datetime(),
});

export type User = z.infer<typeof userSchema>;

const usersResponseSchema = z.object({
  users: z.array(userSchema),
  total: z.number(),
  page: z.number(),
  pageSize: z.number(),
});

export type UsersResponse = z.infer<typeof usersResponseSchema>;

// API functions
export const usersApi = {
  getAll: async (
    page: number = 1,
    limit: number = 10
  ): Promise<UsersResponse> => {
    const { data } = await apiClient.get("/users", {
      params: { page, limit },
    });
    return usersResponseSchema.parse(data);
  },

  getById: async (id: string): Promise<User> => {
    const { data } = await apiClient.get(`/users/${id}`);
    return userSchema.parse(data);
  },

  create: async (userData: Omit<User, "id" | "createdAt">): Promise<User> => {
    const { data } = await apiClient.post("/users", userData);
    return userSchema.parse(data);
  },

  update: async (id: string, userData: Partial<User>): Promise<User> => {
    const { data } = await apiClient.patch(`/users/${id}`, userData);
    return userSchema.parse(data);
  },

  delete: async (id: string): Promise<void> => {
    await apiClient.delete(`/users/${id}`);
  },
};
```

---

### React Query Hooks

**`src/lib/hooks/useUsers.ts`**

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { usersApi, type User, type UsersResponse } from "@/lib/api/users";
import { toast } from "sonner"; // or your preferred toast library

// Query keys for better organization
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  list: (page: number, limit: number) =>
    [...userKeys.lists(), { page, limit }] as const,
  details: () => [...userKeys.all, "detail"] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

// Fetch all users with pagination
export function useUsers(page: number = 1, limit: number = 10) {
  return useQuery({
    queryKey: userKeys.list(page, limit),
    queryFn: () => usersApi.getAll(page, limit),
    keepPreviousData: true, // Show old data while fetching new page
  });
}

// Fetch single user
export function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: !!id, // Only run if id exists
  });
}

// Create user mutation
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: usersApi.create,
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: userKeys.lists() });

      // Snapshot previous value
      const previousUsers = queryClient.getQueryData(userKeys.lists());

      // Optimistically update cache
      queryClient.setQueryData<UsersResponse>(userKeys.list(1, 10), (old) => {
        if (!old) return old;
        return {
          ...old,
          users: [
            { ...newUser, id: "temp-id", createdAt: new Date().toISOString() },
            ...old.users,
          ],
          total: old.total + 1,
        };
      });

      return { previousUsers };
    },
    onError: (err, newUser, context) => {
      // Rollback on error
      queryClient.setQueryData(userKeys.lists(), context?.previousUsers);
      toast.error("Failed to create user");
    },
    onSuccess: (data) => {
      toast.success("User created successfully");
    },
    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}

// Update user mutation
export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
      usersApi.update(id, data),
    onSuccess: (data, variables) => {
      // Update specific user in cache
      queryClient.setQueryData(userKeys.detail(variables.id), data);
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
      toast.success("User updated successfully");
    },
    onError: () => {
      toast.error("Failed to update user");
    },
  });
}

// Delete user mutation
export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: usersApi.delete,
    onSuccess: (_, deletedId) => {
      // Remove from cache
      queryClient.removeQueries({ queryKey: userKeys.detail(deletedId) });
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
      toast.success("User deleted successfully");
    },
    onError: () => {
      toast.error("Failed to delete user");
    },
  });
}
```

---

### Using React Query in Components

**`src/components/features/UsersList.tsx`**

```typescript
"use client";

import { useState } from "react";
import { useUsers, useDeleteUser } from "@/lib/hooks/useUsers";
import { Button } from "@/components/ui/Button";

export function UsersList() {
  const [page, setPage] = useState(1);
  const { data, isLoading, isError, error } = useUsers(page, 10);
  const deleteMutation = useDeleteUser();

  if (isLoading) {
    return <div>Loading users...</div>;
  }

  if (isError) {
    return <div>Error: {error.message}</div>;
  }

  if (!data) {
    return <div>No data</div>;
  }

  return (
    <div className="space-y-4">
      <div className="grid gap-4">
        {data.users.map((user) => (
          <div
            key={user.id}
            className="border rounded-lg p-4 flex justify-between items-center"
          >
            <div>
              <h3 className="font-semibold">{user.name}</h3>
              <p className="text-sm text-gray-600">{user.email}</p>
            </div>
            <Button
              variant="destructive"
              size="sm"
              onClick={() => deleteMutation.mutate(user.id)}
              isLoading={deleteMutation.isLoading}
            >
              Delete
            </Button>
          </div>
        ))}
      </div>

      {/* Pagination */}
      <div className="flex justify-between items-center">
        <Button
          onClick={() => setPage((p) => Math.max(1, p - 1))}
          disabled={page === 1}
        >
          Previous
        </Button>
        <span>
          Page {page} of {Math.ceil(data.total / 10)}
        </span>
        <Button
          onClick={() => setPage((p) => p + 1)}
          disabled={page >= Math.ceil(data.total / 10)}
        >
          Next
        </Button>
      </div>
    </div>
  );
}
```

---

## 2️⃣ Zustand for Client State

### Create Store

**`src/lib/store/useAuthStore.ts`**

```typescript
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";

interface User {
  id: string;
  name: string;
  email: string;
  role: "user" | "admin";
}

interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
}

interface AuthActions {
  login: (user: User, accessToken: string, refreshToken: string) => void;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
  setTokens: (accessToken: string, refreshToken: string) => void;
}

type AuthStore = AuthState & AuthActions;

export const useAuthStore = create<AuthStore>()(
  persist(
    immer((set) => ({
      // Initial state
      user: null,
      accessToken: null,
      refreshToken: null,
      isAuthenticated: false,

      // Actions
      login: (user, accessToken, refreshToken) =>
        set((state) => {
          state.user = user;
          state.accessToken = accessToken;
          state.refreshToken = refreshToken;
          state.isAuthenticated = true;
        }),

      logout: () =>
        set((state) => {
          state.user = null;
          state.accessToken = null;
          state.refreshToken = null;
          state.isAuthenticated = false;
        }),

      updateUser: (updates) =>
        set((state) => {
          if (state.user) {
            state.user = { ...state.user, ...updates };
          }
        }),

      setTokens: (accessToken, refreshToken) =>
        set((state) => {
          state.accessToken = accessToken;
          state.refreshToken = refreshToken;
        }),
    })),
    {
      name: "auth-storage",
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        user: state.user,
        accessToken: state.accessToken,
        refreshToken: state.refreshToken,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);

// Selectors for better performance
export const useUser = () => useAuthStore((state) => state.user);
export const useIsAuthenticated = () =>
  useAuthStore((state) => state.isAuthenticated);
export const useAccessToken = () => useAuthStore((state) => state.accessToken);
```

---

### UI State Store

**`src/lib/store/useUIStore.ts`**

```typescript
import { create } from "zustand";

interface UIState {
  isSidebarOpen: boolean;
  theme: "light" | "dark" | "system";
  notifications: Array<{
    id: string;
    message: string;
    type: "success" | "error" | "info";
  }>;
  toggleSidebar: () => void;
  setTheme: (theme: "light" | "dark" | "system") => void;
  addNotification: (
    message: string,
    type: "success" | "error" | "info"
  ) => void;
  removeNotification: (id: string) => void;
}

export const useUIStore = create<UIState>((set) => ({
  isSidebarOpen: true,
  theme: "system",
  notifications: [],

  toggleSidebar: () =>
    set((state) => ({ isSidebarOpen: !state.isSidebarOpen })),

  setTheme: (theme) => set({ theme }),

  addNotification: (message, type) =>
    set((state) => ({
      notifications: [
        ...state.notifications,
        { id: Date.now().toString(), message, type },
      ],
    })),

  removeNotification: (id) =>
    set((state) => ({
      notifications: state.notifications.filter((n) => n.id !== id),
    })),
}));
```

---

### Using Zustand in Components

**`src/components/layouts/Sidebar.tsx`**

```typescript
"use client";

import { useUIStore } from "@/lib/store/useUIStore";
import { useUser } from "@/lib/store/useAuthStore";

export function Sidebar() {
  const isSidebarOpen = useUIStore((state) => state.isSidebarOpen);
  const toggleSidebar = useUIStore((state) => state.toggleSidebar);
  const user = useUser();

  if (!isSidebarOpen) return null;

  return (
    <aside className="w-64 bg-gray-100 p-4 h-screen">
      <div className="flex justify-between items-center mb-4">
        <h2 className="text-xl font-bold">Menu</h2>
        <button onClick={toggleSidebar}>✕</button>
      </div>

      {user && (
        <div className="mb-4 p-3 bg-white rounded">
          <p className="font-semibold">{user.name}</p>
          <p className="text-sm text-gray-600">{user.email}</p>
        </div>
      )}

      <nav>
        <ul className="space-y-2">
          <li>Dashboard</li>
          <li>Settings</li>
          <li>Profile</li>
        </ul>
      </nav>
    </aside>
  );
}
```

---

## 🎨 Best Practices

### 1. Separate Server and Client State

```typescript
// ✅ GOOD: Server state with React Query
const { data: users } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
});

// ✅ GOOD: Client state with Zustand
const isSidebarOpen = useUIStore((state) => state.isSidebarOpen);

// ❌ BAD: Mixing concerns
const [users, setUsers] = useState([]); // Should use React Query
```

### 2. Use Query Keys Consistently

```typescript
// ✅ GOOD: Organized query keys
export const queryKeys = {
  users: {
    all: ["users"] as const,
    lists: () => [...queryKeys.users.all, "list"] as const,
    list: (filters: string) =>
      [...queryKeys.users.lists(), { filters }] as const,
    details: () => [...queryKeys.users.all, "detail"] as const,
    detail: (id: string) => [...queryKeys.users.details(), id] as const,
  },
};

// ❌ BAD: Inconsistent keys
useQuery({ queryKey: ["users"] });
useQuery({ queryKey: ["user", id] });
useQuery({ queryKey: ["usersList"] });
```

### 3. Optimistic Updates

```typescript
// ✅ GOOD: Optimistic update with rollback
const mutation = useMutation({
  mutationFn: updateUser,
  onMutate: async (newData) => {
    await queryClient.cancelQueries(["user", id]);
    const previous = queryClient.getQueryData(["user", id]);
    queryClient.setQueryData(["user", id], newData);
    return { previous };
  },
  onError: (err, newData, context) => {
    queryClient.setQueryData(["user", id], context?.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries(["user", id]);
  },
});
```

### 4. Proper Error Handling

```typescript
// ✅ GOOD: Comprehensive error handling
const { data, isLoading, isError, error } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
  retry: 3,
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
});

if (isError) {
  if (axios.isAxiosError(error)) {
    if (error.response?.status === 404) {
      return <NotFound />;
    }
    return <Error message={error.response?.data?.message} />;
  }
  return <Error message="An unexpected error occurred" />;
}
```

---

## 🚀 Advanced Patterns

### Infinite Scrolling

```typescript
import { useInfiniteQuery } from "@tanstack/react-query";

function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam = 1 }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage, pages) => {
      return lastPage.hasMore ? pages.length + 1 : undefined;
    },
  });
}

function PostsList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = usePosts();

  return (
    <div>
      {data?.pages.map((page) =>
        page.posts.map((post) => <PostCard key={post.id} post={post} />)
      )}
      {hasNextPage && (
        <Button onClick={() => fetchNextPage()} isLoading={isFetchingNextPage}>
          Load More
        </Button>
      )}
    </div>
  );
}
```

### Dependent Queries

```typescript
function UserPosts({ userId }: { userId: string }) {
  // First query
  const { data: user } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });

  // Second query depends on first
  const { data: posts } = useQuery({
    queryKey: ["posts", user?.id],
    queryFn: () => fetchUserPosts(user!.id),
    enabled: !!user, // Only run when user exists
  });

  return <div>{/* Render posts */}</div>;
}
```

---

## 📚 Next Steps

- [Rendering Strategies →](./03-rendering-strategies.md)
- [Performance Optimization →](./04-performance-optimization.md)

---

**🎯 Key Takeaway:** Use React Query for server state (API data) and Zustand for client state (UI state). This separation provides better caching, automatic refetching, and cleaner code architecture.
