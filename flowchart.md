# Campus GPT Frontend — Component Hierarchy & Execution Flowcharts

This document details the complete component hierarchy, client-side routing trees, execution flowcharts, and asynchronous state lifecycles for the **Campus GPT Frontend**. It traces data and event execution across Next.js 15 App Router layouts, React 19 Client Components, TanStack Query caches, and Supabase Authentication listeners.

---

## 1. Global Application Hierarchy & Root Provider Pipeline

Every page rendered in Campus GPT is encapsulated by a cascaded hierarchy of context providers defined in `src/app/layout.tsx`.

```mermaid
flowchart TD
    RootHTML["Root HTML Document (src/app/layout.tsx)"] --> TanStack["TanStackProvider (src/providers/tanstack.provider.tsx)"]
    TanStack --> Supabase["SupabaseProvider (src/providers/supabse.provider.tsx)"]
    Supabase --> Axios["AxiosProvider (src/services/axios/axios.provider.tsx)"]
    Axios --> Toast["Toaster Notification Mount (src/components/ui/sonner.tsx)"]
    Toast --> RouterOutlet["Next.js App Router Page Outlet"]

    RouterOutlet --> RouteCheck{"Evaluate Pathname"}
    
    RouteCheck -->|"/auth/*"| AuthLayout["AuthLayout (src/app/auth/layout.tsx)"]
    AuthLayout --> LoginView["/auth/login"]
    AuthLayout --> SignupView["/auth/signup"]
    
    RouteCheck -->|"/(student)/*"| StudentLayout["StudentLayout (src/app/(student)/layout.tsx)"]
    StudentLayout --> StudentNav["StudentNav Header (src/app/(student)/dashboard/partials/student-nav.tsx)"]
    StudentNav --> StudentViews["Student Pages: /dashboard, /chatbot, /timetable, /select-course"]
    
    RouteCheck -->|"/admin/*"| AdminLayout["AdminLayout (src/app/admin/layout.tsx)"]
    AdminLayout --> AdminNav["AdminNav Header (src/app/admin/partials/admin-nav.tsx)"]
    AdminNav --> AdminViews["Admin Pages: /admin, /admin/courses, /admin/faculty, /admin/building, /admin/events"]
```

---

## 2. Authentication & Session Initialization Lifecycle

The authentication lifecycle handles credential validation, token propagation into HTTP interceptors, and automated role-based navigation.

```mermaid
sequenceDiagram
    autonumber
    actor User as Student / Administrator
    participant UI as Login Page (src/app/auth/login/page.tsx)
    participant Supabase as Supabase Client (src/lib/supabse.ts)
    participant AuthContext as SupabaseProvider (src/providers/supabse.provider.tsx)
    participant AxiosContext as AxiosProvider (src/services/axios/axios.provider.tsx)
    participant Router as Next.js App Router

    User->>UI: Enters Email & Password (or clicks Google OAuth)
    UI->>Supabase: supabase.auth.signInWithPassword({ email, password })
    
    alt Credentials Invalid
        Supabase-->>UI: Return AuthError
        UI->>UI: Display error message via Sonner toast
    else Credentials Valid
        Supabase-->>UI: Return Session { access_token, user }
        UI->>AuthContext: onAuthStateChange triggered with 'SIGNED_IN'
        AuthContext->>AuthContext: Set user & session state
        
        AuthContext->>AxiosContext: Provide active access_token
        AxiosContext->>AxiosContext: Configure Axios default header: Authorization = Bearer <access_token>
        
        UI->>Router: Check user.role
        alt Role is Admin
            Router->>Router: router.push('/admin')
        else Role is Student
            Router->>Router: router.push('/dashboard')
        end
    end
```

---

## 3. Student Route Guard & Focus Persistence Flow

The student layout enforces authentication while preserving the student's active route across window focus events.

```mermaid
flowchart TD
    NavStudent["Student navigates to a (student)/* route"] --> GuardCheck["(student)/layout.tsx evaluates session"]
    
    GuardCheck --> CheckLoading{"Is Auth State Loading?"}
    CheckLoading -->|"Yes"| ShowSkeleton["Render Loading Spinner Shell"]
    CheckLoading -->|"No"| CheckUser{"Is user authenticated?"}
    
    CheckUser -->|"No"| RedirectLogin["router.push('/auth/login')"]
    CheckUser -->|"Yes"| CheckRole{"Is user.role === 'admin'?"}
    
    CheckRole -->|"Yes"| RedirectAdmin["router.push('/admin')"]
    CheckRole -->|"No (Student)"| CheckPath{"Pathname Check"}
    
    CheckPath --> RenderActive["Render Target Component Outlet (e.g. /timetable, /chatbot)"]
    
    subgraph Window_Focus_Stability["Tab-Focus Resilience"]
        RenderActive -.->|"User switches browser tabs and returns"| WindowFocus["Window Focus Event Triggered"]
        WindowFocus -.-> Recheck["Dependencies evaluated in layout.tsx"]
        Recheck -.-> Preserved["Pathname preserved: Does NOT reset to /dashboard"]
    end
```

---

## 4. Student Dashboard & Timetable Data Fetching Flow

The student dashboard orchestrates multiple parallel server-state queries via TanStack Query to construct the student's day agenda.

```mermaid
sequenceDiagram
    autonumber
    actor Student
    participant Page as DashboardPage (src/app/(student)/dashboard/page.tsx)
    participant Provider as DashboardProvider (src/app/(student)/dashboard/dashboard.provider.tsx)
    participant API as dashboard.api.ts
    participant Axios as Axios Context Instance
    participant Backend as Campus GPT Backend API

    Student->>Page: Navigates to /dashboard
    Page->>Provider: Mounts and invokes useDashboard()
    
    par Fetch Enrolled Course
        Provider->>API: fetchStudentCourse()
        API->>Axios: GET /education/course/joined
        Axios->>Backend: HTTP GET /education/course/joined
        Backend-->>Axios: 200 OK: { data: { course: MCA, id: "..." } }
        Axios-->>Provider: Enrolled Course Entity
    and Fetch Today's Schedule
        Provider->>API: fetchStudentTodaySchedule(courseId)
        API->>Axios: GET /education/course-timetable?courseId=...
        Axios->>Backend: HTTP GET /education/course-timetable
        Backend-->>Axios: 200 OK: { data: [ { subject, faculty, building, startTime } ] }
        Axios-->>Provider: Today's Class Slots
    and Fetch University Events
        Provider->>API: fetchUpcomingEvents()
        API->>Axios: GET /events
        Axios->>Backend: HTTP GET /events
        Backend-->>Axios: 200 OK: { data: [ { title, eventDate, location } ] }
        Axios-->>Provider: Upcoming Events List
    end

    Provider->>Page: Deliver unified state { course, todayClasses, events, isLoading: false }
    Page->>Student: Render Hero Stats Card, Active Class Badge & Schedule Timeline
```

---

## 5. Course Enrollment Lifecycle (`/select-course`)

When a new student joins the platform, they must pick their degree program before accessing customized timetables.

```mermaid
flowchart TD
    OpenSelect["Student opens /select-course"] --> FetchCourses["useQuery: fetchAllCourses()"]
    FetchCourses --> DisplayGrid["Render Degree Program Cards (MCA, BCA, B.Tech, MBA)"]
    
    DisplayGrid --> ClickEnroll["Student clicks 'Enroll in Course'"]
    ClickEnroll --> MutateJoin["useMutation: joinCourse({ courseId })"]
    
    MutateJoin --> PostAPI["Axios POST /education/course/join"]
    PostAPI --> BackendRes{"Backend Response"}
    
    BackendRes -->|"Success (201)"| InvalidateCache["queryClient.invalidateQueries(['student-course'])"]
    InvalidateCache --> NotifyUser["Sonner.toast.success('Successfully enrolled!')"]
    NotifyUser --> RouteTimetable["router.push('/timetable')"]
    
    BackendRes -->|"Error"| NotifyError["Sonner.toast.error('Enrollment failed')"]
```

---

## 6. AI Concierge Conversational Pipeline (`/chatbot/[chatid]`)

The chatbot UI delivers a real-time conversational feed with optimistic updates, animated loading indicators, and markdown formatting.

```mermaid
sequenceDiagram
    autonumber
    actor Student
    participant ChatUI as ChatRoomView (src/app/(student)/chatbot/[chatid]/page.tsx)
    participant Provider as ChatbotProvider (src/app/(student)/chatbot/chatbot.provider.tsx)
    participant API as chatbot.api.tsx
    participant Backend as Backend /chat/message
    participant BottomAnchor as ScrollArea End Ref

    Student->>ChatUI: Types prompt: "Where is my Database lab today?"
    Student->>ChatUI: Presses Enter / Clicks Send Button
    
    rect rgb(245, 255, 250)
        Note over ChatUI,Provider: Optimistic UI Message Append
        ChatUI->>Provider: appendUserMessage(prompt)
        Provider->>ChatUI: Instantly render student message bubble
        ChatUI->>BottomAnchor: scrollIntoView({ behavior: 'smooth' })
        ChatUI->>ChatUI: Set isTyping = true (Renders glowing typing indicator)
    end

    Provider->>API: sendMessage({ chatId, content: prompt })
    API->>Backend: POST /chat/message { chatId, content }
    
    Note over Backend: Database context grounding + Gemini 3.5 Flash generation
    
    Backend-->>API: 200 OK: { data: { role: 'assistant', content: botReply } }
    API-->>Provider: Deliver Assistant Response
    
    rect rgb(240, 248, 255)
        Note over Provider,ChatUI: Render AI Answer
        Provider->>ChatUI: Set isTyping = false
        Provider->>ChatUI: Append assistant message bubble
        ChatUI->>ChatUI: Parse Markdown (bold, lists, anchor link buttons for map coordinates)
        ChatUI->>BottomAnchor: scrollIntoView({ behavior: 'smooth' })
    end
```

---

## 7. Campus Building & Location Management Flow (`/admin/building`)

Administrators configure physical campus blocks with coordinates and navigation directions.

```mermaid
flowchart TD
    AdminOpen["Admin opens /admin/building"] --> FetchBuildings["useQuery: fetchAllBuildings()"]
    FetchBuildings --> RenderList["Display Buildings Table (Block Code, Name, Coordinates)"]
    
    RenderList --> ClickAdd["Admin clicks 'Add New Building'"]
    ClickAdd --> OpenDialog["Open LocationManager Dialog (src/app/admin/building/partials/location-manager.tsx)"]
    
    OpenDialog --> MapPicker["MapSelector (src/app/admin/building/partials/map-selector.tsx)"]
    MapPicker --> SelectCoords["Admin selects point on Map or enters Lat/Lng"]
    
    SelectCoords --> FillForm["BuildingForm: Enter Name ('Block 34'), Code ('B34'), Description"]
    FillForm --> SubmitBuilding["Submit BuildingForm"]
    
    SubmitBuilding --> MutateCreate["useMutation: createBuilding(dto)"]
    MutateCreate --> PostBuilding["Axios POST /building"]
    
    PostBuilding --> CreateRes{"Create Status"}
    CreateRes -->|"Success"| InvalidateBuildings["queryClient.invalidateQueries(['buildings'])"]
    InvalidateBuildings --> CloseDialog["Close Dialog Modal"]
    CloseDialog --> ToastSuccess["Sonner.toast.success('Building created!')"]
    ToastSuccess --> RenderList
    
    CreateRes -->|"Error"| ToastFail["Sonner.toast.error('Building creation failed')"]
```

---

## 8. Global HTTP Interceptor & Error Handling Flow

```mermaid
flowchart TD
    HttpRequest["Component executes Axios API Call"] --> InterceptorReq["Axios Request Interceptor"]
    
    InterceptorReq --> AttachAuth["Read session.access_token from SupabaseContext"]
    AttachAuth --> SendHeader["Set headers['Authorization'] = 'Bearer ' + token"]
    SendHeader --> NetworkCall["Execute Outgoing HTTP Call"]
    
    NetworkCall --> ResponseArrives{"HTTP Response Status"}
    
    ResponseArrives -->|"200 / 201 OK"| UnpackEnvelope["Unwrap response.data.data payload"]
    UnpackEnvelope --> ReturnToComponent["Deliver typed result to React Query"]
    
    ResponseArrives -->|"401 Unauthorized"| Handle401["Clear stale user session"]
    Handle401 --> RedirectToLogin["router.push('/auth/login')"]
    Handle401 --> ShowToastAuth["Sonner.toast.error('Session expired, please login again')"]
    
    ResponseArrives -->|"403 Forbidden"| ShowToast403["Sonner.toast.error('You do not have permission for this action')"]
    
    ResponseArrives -->|"400 / 500 Error"| ExtractMessage["Extract error message from meta.message / failures"]
    ExtractMessage --> ShowToastErr["Sonner.toast.error(errorMessage || 'Something went wrong')"]
    ShowToastErr --> RejectPromise["Throw error to trigger React Query isError state"]
```
