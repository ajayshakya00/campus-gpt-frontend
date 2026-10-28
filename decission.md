# Campus GPT Frontend — Architecture Decision Records (ADR) & Technical Rationale

This document serves as the official Architecture Decision Record (ADR) and engineering rationale for the **Campus GPT Frontend**. It details the architectural paradigms, UI/UX trade-offs, state management strategies, library choices, and client-side performance optimizations powering the student and administrator web experience.

---

## 1. Executive Summary & Design Vision

The Campus GPT Frontend is designed as a responsive, lightning-fast academic portal and AI concierge interface. It delivers two distinct user experiences:
1. **Student Companion Portal**:
   - Personalized daily academic agenda (today's active classes, room numbers, faculty).
   - Conversational AI assistant with persistent conversation threads and markdown-rendered campus directions.
   - Interactive weekly timetable schedule with day-by-day course inspection.
   - One-click degree program enrollment.
2. **University Administration Suite**:
   - Real-time institutional KPIs (active enrollments, faculty counts, course numbers).
   - Degree program and subject curriculum creation.
   - Batch timetable scheduler with conflict prevention.
   - Campus building directory with Google Maps geolocation coordinate selection.
   - Campus events calendar publisher.

---

## 2. Core Framework & Language Decisions

### 2.1 Framework: Next.js 15 (App Router with Turbopack)

- **Decision**: Build the application using Next.js 15 (`app/` directory) and compile with Turbopack (`next build --turbopack`).
- **Alternatives Considered**: Create React App, Vite SPA, Remix / React Router v7.
- **Rationale & Advantages**:
  - **App Router Layout Hierarchy**: Nested layouts (`(student)/layout.tsx`, `admin/layout.tsx`, `auth/layout.tsx`) allow persistent navigations, sidebars, and route-protection wrappers without re-rendering the outer shell during page transitions.
  - **Route Groups**: Parenthesized route groups (e.g. `(student)`) decouple file-system URL paths from directory structures, keeping student routes clean (`/dashboard`, `/chatbot`, `/timetable`) while sharing authenticated student layout state.
  - **Turbopack Build Speed**: Turbopack delivers 10x faster local Fast Refresh and high-speed production compilation, ensuring rapid developer iteration.
  - **Zero-Config Vercel Edge Integration**: Automatic optimization of static assets, dynamic API routes (`/api/auth/session`), and seamless preview deployments.

### 2.2 Language: TypeScript 5.x (Strict Mode)

- **Decision**: Strict TypeScript across all components, hooks, API callers, and providers.
- **Rationale & Advantages**:
  - Enforces end-to-end type contracts matching the backend API response structures (`ApiResponse<T>`, `Course`, `TimetableSlot`, `Faculty`, `Building`).
  - Prevents runtime component crashes caused by nullable nested properties (e.g. `slot.subject?.name` or `slot.building?.code`).

### 2.3 Runtime: React 19

- **Decision**: Utilize React 19 capabilities.
- **Rationale & Advantages**:
  - Native optimization for concurrent features, smooth client transitions, and modern state batching.

---

## 3. Styling & Component Design System

### 3.1 Styling Engine: Tailwind CSS v4

- **Decision**: Adopt Tailwind CSS v4 with CSS-first variable themes (`globals.css`).
- **Alternatives Considered**: Emotion / Styled-Components, CSS Modules, Tailwind v3.
- **Rationale & Advantages**:
  - **Zero Build-Time CSS Overhead**: Tailwind v4 uses a lightning-fast Rust-based compiler with no legacy PostCSS configuration dependencies.
  - **Theme Modernization**: Direct support for modern CSS variables (`--color-background`, `--color-foreground`, modern OKLCH color spaces), enabling crisp dark mode aesthetics and glowing futuristic accents suitable for Campus GPT.
  - **Production Dynamic Gradient Bug Resolution**: Resolved invisible metric numbers on stat cards caused by uncompiled arbitrary dynamic Tailwind class names by standardizing on pre-compiled explicit gradient utility tokens (`text-white font-extrabold`).

### 3.2 UI Primitives: shadcn/ui & Radix UI

- **Decision**: Implement accessible, headless UI primitives via Radix UI wrapped in shadcn/ui components (`Button`, `Card`, `Dialog`, `Select`, `Input`, `Label`, `Badge`, `ScrollArea`, `AlertDialog`).
- **Alternatives Considered**: Material UI (MUI), Ant Design, Chakra UI.
- **Rationale & Advantages**:
  - **Zero Vendor Lock-In**: Code is copied directly into `src/components/ui/`, granting complete engineering ownership to modify styling, animations, or keyboard behaviors.
  - **Accessibility by Default**: Radix UI handles complex ARIA roles, focus traps, keyboard navigation (Escape to close, arrow key selection), and screen-reader announcements out of the box.
  - **Minimal Bundle Footprint**: Only the exact primitives imported in a given route are bundled into that page's chunk.

### 3.3 Toast Notifications: Sonner

- **Decision**: Adopt `sonner` via `src/components/ui/sonner.tsx`.
- **Alternatives Considered**: React-Toastify, React-Hot-Toast.
- **Rationale & Advantages**:
  - Modern stacked toast UI with fluid animations, auto-dismiss, action buttons, and full dark mode synchronization.

---

## 4. State Management & Data Fetching Architecture

### 4.1 Server-State Caching: TanStack Query (React Query v5)

- **Decision**: Manage all asynchronous server data using `@tanstack/react-query`.
- **Alternatives Considered**: Redux Toolkit, Zustand, SWR, manual `useEffect` + `useState`.
- **Rationale & Advantages**:
  - **Automatic Cache Invalidation & Refetching**: When an administrator creates a new course or building, calling `queryClient.invalidateQueries({ queryKey: ['courses'] })` immediately updates all dependent dropdowns and tables across the UI without manual state synchronization.
  - **Stale-While-Revalidate (SWR)**: Navigating between tabs (`/dashboard` -> `/timetable` -> `/dashboard`) instantly renders cached data from memory before background re-validating, delivering an instantaneous SPA feel.
  - **Built-In Query State Lifecycle**: Exposes declarative booleans (`isPending`, `isError`, `data`, `refetch`), eliminating repetitive boilerplate for loading spinners and error states.

### 4.2 Centralized HTTP Transport: Axios Context (`src/services/axios/`)

- **Decision**: Encapsulate HTTP transport inside an `AxiosProvider` React context:
  ```typescript
  // src/services/axios/axios.provider.tsx
  const axiosInstance = axios.create({ baseURL: process.env.NEXT_PUBLIC_API_URL });
  ```
- **Rationale & Advantages**:
  - **Dynamic JWT Injection**: Automatically attaches `Authorization: Bearer <token>` from the active Supabase session to every outgoing request.
  - **Centralized Error Interceptor**: Intercepts HTTP 401/403/500 errors globally and displays user-friendly Sonner toast notifications without duplicating error logic across 20+ page components.
  - **Custom React Hook**: `useAxios()` exposes the configured instance across both student and admin feature providers.

---

## 5. Authentication & Session Architecture

### 5.1 Provider: Supabase Auth

- **Decision**: Use Supabase Auth with `@supabase/supabase-js`.
- **Alternatives Considered**: NextAuth / Auth.js, Clerk, Firebase Auth, Custom JWT cookies.
- **Rationale & Advantages**:
  - **Direct PostgreSQL Integration**: Supabase Auth securely stores student and admin credentials directly in the managed PostgreSQL instance, enabling seamless foreign keys to university user records.
  - **OAuth & Credentials Support**: Supports traditional email/password credentials alongside one-click Google OAuth sign-in.
  - **Real-Time Auth State Listener**: `supabase.auth.onAuthStateChange()` automatically updates user context across tabs and handles token refreshes transparently.

### 5.2 Streamlined Onboarding UX (Removal of Email Confirmation Barrier)

- **Decision**: Modify the signup flow in `src/app/auth/signup/page.tsx` to automatically authenticate the student and navigate directly to `/dashboard` immediately after successful registration.
- **Rationale & Advantages**:
  - **Frictionless First-Run Experience**: Eliminates the "Check your email inbox to verify" holding screen, reducing drop-off during campus demos and pilot rollouts.
  - **Route Redundancy Cleanup**: Deprecated the redundant `/login` route by configuring a permanent 308 redirect in `next.config.ts` forwarding `/login` to `/auth/login`.

### 5.3 Route Protection & Unwanted Redirect Fix (`src/app/(student)/layout.tsx`)

- **Decision**: Implement role-aware route guards in layout components:
  - If unauthenticated, redirect to `/auth/login`.
  - If authenticated with `role: 'admin'`, redirect from student views to `/admin`.
- **The Critical Focus-Loss Bug Fixed**:
  - Previously, switching browser tabs from `/chatbot` to another window and returning triggered an unexpected redirect back to `/dashboard`.
  - **Root Cause**: The `useEffect` inside `(student)/layout.tsx` was re-evaluating on browser window focus events and triggering `router.push('/dashboard')` whenever user state refreshed.
  - **The Fix**: Stabilized the effect dependencies to only check authentication validity and prevent resetting the student's active pathname when switching tabs.

---

## 6. AI Conversational Interface & UX Design

### 6.1 Conversational State Machine (`src/app/(student)/chatbot/[chatid]/page.tsx`)

- **Decision**: Implement an optimistic, multi-thread conversational UI:
  - Appends user messages immediately to the transcript.
  - Shows an animated typing indicator while awaiting the backend Gemini response.
  - Persists chat room history in PostgreSQL with optional client-side local caching (`src/lib/chat-storage.ts`).
- **Rationale & Advantages**:
  - Delivers snappy, conversational feedback matching modern AI applications (like ChatGPT or Gemini).
  - Auto-scrolls to the newest message smoothly via a bottom ref anchor.
  - Renders markdown with support for bold text, lists, and actionable campus navigation links.

---

## 7. Comprehensive Frontend Dependency Rationale

| Package | Version | Purpose | Advantage / Rationale |
| :--- | :--- | :--- | :--- |
| `next` | `15.5.9` | Full-stack React framework | App Router layouts, Turbopack high-speed compiler, edge asset optimization. |
| `react` & `react-dom` | `19.0.0` | Core UI library | Concurrent rendering, modern transitions, and optimized reconciliation. |
| `@tanstack/react-query` | `^5.64.2` | Server-state caching & sync | Auto-caching, optimistic updates, and declarative query lifecycle states. |
| `@supabase/supabase-js` | `^2.48.0` | Authentication & DB SDK | Managed student auth, JWT sessions, and Google OAuth login. |
| `axios` | `^1.7.9` | HTTP client | Centralized interceptors for Bearer token injection and global error toasts. |
| `tailwindcss` | `^4.0.0` | Utility-first styling engine | CSS-variable native theming, zero build delay, and modern responsive layouts. |
| `@radix-ui/*` primitives | Latest | Headless accessible UI | Keyboard accessibility, ARIA compliance, and unstyled customizable primitives. |
| `lucide-react` | `^0.473.0` | Iconography suite | Clean, modern feather icons for navigation, timetable slots, and campus buildings. |
| `sonner` | `^1.7.3` | Toast notifications | Fluid stacked toast system for API success and error feedback. |
| `clsx` & `tailwind-merge` | `^2.1.1` / `^3.0.1` | Class utility (`cn`) | Safe conditional merging of Tailwind utility classes without specificity conflicts. |
| `dayjs` | `^1.11.13` | Date formatting | Lightweight date calculations for timetable days and upcoming campus events. |
