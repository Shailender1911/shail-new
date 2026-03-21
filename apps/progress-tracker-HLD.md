# High-Level Design (HLD) — Interview Preparation Tracker

**Version:** 1.0  
**Date:** March 2026  
**Author:** Shailender Kumar  

---

## 1. Introduction

### 1.1 Purpose
The Interview Preparation Tracker is a full-stack web application designed to help software engineers systematically manage and track their interview preparation across multiple technical domains. It addresses the common problem of fragmented study planning, inconsistent tracking, and lack of visibility into preparation progress.

### 1.2 Scope
The MVP covers:
- User authentication and multi-tenant data isolation
- Hierarchical study topic management with progress tracking
- Daily task planning with priority management
- Dashboard with aggregated analytics

### 1.3 Target Users
- Software Engineers (3-8 YOE) preparing for job switches
- Engineers preparing across multiple domains (DSA, System Design, Language-specific, Behavioral)

---

## 2. System Architecture

### 2.1 Architecture Style
**3-Tier Layered Architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────┐
│                    CLIENT TIER                       │
│                                                     │
│   React 19 + TypeScript + Tailwind CSS v4           │
│   ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│   │ React    │ │ React    │ │ Zod + React Hook  │  │
│   │ Router   │ │ Query v5 │ │ Form              │  │
│   └──────────┘ └──────────┘ └───────────────────┘  │
│         │            │               │              │
│         └────────────┼───────────────┘              │
│                      │                              │
│              Axios HTTP Client                      │
│              (JWT Bearer Token)                     │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS / REST
                       │ /api/v1/*
┌──────────────────────┼──────────────────────────────┐
│                 APPLICATION TIER                     │
│                      │                              │
│   Spring Boot 3.3 + Java 17                        │
│   ┌──────────────────┼─────────────────────────┐   │
│   │     Security Layer (JWT Filter Chain)       │   │
│   ├─────────────────────────────────────────────┤   │
│   │     Controller Layer (REST Endpoints)       │   │
│   │  ┌──────────┐ ┌───────┐ ┌──────┐ ┌──────┐  │   │
│   │  │ Auth     │ │ Topic │ │ Todo │ │ Dash │  │   │
│   │  │Controller│ │Ctrl   │ │Ctrl  │ │Ctrl  │  │   │
│   │  └──────────┘ └───────┘ └──────┘ └──────┘  │   │
│   ├─────────────────────────────────────────────┤   │
│   │     Service Layer (Business Logic)          │   │
│   │  ┌──────────┐ ┌────────────┐ ┌───────────┐ │   │
│   │  │ Auth     │ │ TopicTree  │ │ Todo      │ │   │
│   │  │ Service  │ │ Service    │ │ Service   │ │   │
│   │  └──────────┘ └────────────┘ └───────────┘ │   │
│   ├─────────────────────────────────────────────┤   │
│   │     Repository Layer (Data Access)          │   │
│   │  ┌──────────┐ ┌───────────┐ ┌────────────┐ │   │
│   │  │ JPA Repo │ │ JPA Spec  │ │ Flyway     │ │   │
│   │  │          │ │ (Filters) │ │ Migrations │ │   │
│   │  └──────────┘ └───────────┘ └────────────┘ │   │
│   └─────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────┘
                       │ JDBC
┌──────────────────────┼──────────────────────────────┐
│                    DATA TIER                         │
│                      │                              │
│   PostgreSQL 15                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │ users │ categories │ topics │ todos │ tags  │   │
│   │       │            │        │       │       │   │
│   │       │ study_sessions │ entity_tags         │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 2.2 Key Architecture Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Monolithic (layered) | MVP scope doesn't justify microservices overhead |
| API Style | REST (JSON) | Standard, well-tooled, fits CRUD-heavy domain |
| Auth | JWT (stateless) | No session server needed, scales horizontally |
| ORM | Hibernate (JPA) | Spring ecosystem standard, strong type safety |
| Migrations | Flyway | Version-controlled schema, repeatable deployments |
| Frontend State | React Query (server state) | Cache invalidation, optimistic updates, parallel loading |
| Validation | Dual (Zod + Bean Validation) | Identical constraints on client and server |

---

## 3. Component Design

### 3.1 Frontend Components

```
Frontend Architecture
├── Routing Layer (React Router v7)
│   ├── Public Routes: /login, /register
│   └── Protected Routes (requires JWT)
│       ├── /dashboard  → DashboardPage
│       ├── /topics     → TopicsPage
│       └── /planner    → DailyPlannerPage
│
├── State Management
│   ├── AuthContext     → JWT tokens, user info, login/logout
│   ├── ThemeContext    → Dark/light mode toggle
│   └── React Query    → Server state (categories, topics, todos, dashboard)
│
├── Service Layer (Axios)
│   ├── api.ts          → Base instance, JWT interceptor, 401 refresh
│   ├── authService     → login, register, refresh
│   ├── categoryService → CRUD categories
│   ├── topicService    → CRUD topics with tree support
│   ├── todoService     → CRUD todos, complete action
│   └── dashboardService → 3 parallel aggregation endpoints
│
└── UI Component Library (14 shadcn-style components)
    ├── Button, Input, Label, Textarea
    ├── Card, Badge, Progress, Skeleton
    ├── Dialog, Select, DropdownMenu
    └── Sheet, Separator, Tooltip
```

### 3.2 Backend Components

```
Backend Architecture
├── Security Module
│   ├── JwtTokenProvider      → Token generation (HS384), validation, expiry
│   ├── JwtAuthenticationFilter → OncePerRequestFilter, extracts user from token
│   ├── SecurityConfig        → Filter chain, CORS, endpoint permissions
│   └── UserDetailsServiceImpl → Loads user by username for Spring Security
│
├── Controller Module (REST API)
│   ├── AuthController        → POST /auth/register, /login, /refresh
│   ├── CategoryController    → GET, POST, PATCH /categories
│   ├── TopicController       → GET, POST, PATCH, DELETE /topics
│   ├── TodoController        → GET, POST, PATCH, DELETE /todos
│   └── DashboardController   → GET /dashboard/{today-tasks,category-progress,weekly-summary}
│
├── Service Module (Business Logic)
│   ├── AuthServiceImpl       → Registration (BCrypt), login (JWT issuance)
│   ├── CategoryServiceImpl   → CRUD + computed progress from child topics
│   ├── TopicTreeServiceImpl  → Tree operations, depth validation, status transitions
│   ├── TodoServiceImpl       → Specification-based filtering, completion workflow
│   └── DashboardServiceImpl  → Aggregation queries (no denormalized fields)
│
├── Repository Module (Data Access)
│   ├── JPA Repositories      → Standard CRUD + custom user-scoped queries
│   ├── TodoSpecification     → Dynamic predicate builder (Open/Closed Principle)
│   └── Flyway Migrations     → V1-V7 versioned schema DDL + seed data
│
└── Cross-cutting
    ├── GlobalExceptionHandler → RFC 7807 error responses
    ├── BaseEntity            → id, createdAt, updatedAt, version (optimistic locking)
    └── DTO Layer             → Request/Response separation (no entity leakage)
```

---

## 4. Data Model

### 4.1 Entity Relationship Diagram

```
┌──────────────┐
│    users     │
│──────────────│
│ PK id        │
│ username (UK)│
│ email (UK)   │
│ password_hash│
│ display_name │
│ version      │
└──────┬───────┘
       │ 1:N (user_id FK on all tables)
       │
  ┌────┴────┬──────────┬──────────┬─────────┐
  │         │          │          │         │
  ▼         ▼          ▼          ▼         ▼
┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌──────┐
│categ.│ │topics│ │todos │ │study_  │ │tags  │
│      │ │      │ │      │ │sessions│ │      │
└──┬───┘ └──┬───┘ └──────┘ └────────┘ └──┬───┘
   │        │                             │
   │ 1:N    │ self-ref (parent_topic_id)  │ 1:N
   │        │ topics → topics             │
   └────────┤                             │
   category │ N:1                    ┌────┴────┐
   has many │ todo → topic           │entity_  │
   topics   │ session → topic        │tags     │
            │                        │(polymor)│
            └────────────────────────└─────────┘
```

### 4.2 Key Constraints

| Table | Constraint | Purpose |
|-------|-----------|---------|
| topics | `CHECK (depth <= 2)` | Max 3-level hierarchy (Category > Topic > Subtopic) |
| topics | `CHECK (status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED'))` | Enum-safe status |
| todos | `CHECK (priority IN ('HIGH','MEDIUM','LOW'))` | Enum-safe priority |
| todos | `CHECK (status IN ('PENDING','IN_PROGRESS','COMPLETED'))` | Enum-safe status |
| all tables | `version INTEGER` | Optimistic locking (prevents lost updates) |
| all tables | `user_id NOT NULL` | Multi-tenant data isolation |

### 4.3 Indexing Strategy

```sql
-- Hot path: user's topics by category
CREATE INDEX idx_topics_user_category ON topics(user_id, category_id);

-- Hot path: user's todos by date (daily planner default view)
CREATE INDEX idx_todos_user_date ON todos(user_id, due_date);

-- Todo filtering
CREATE INDEX idx_todos_status ON todos(status);

-- Study session aggregation
CREATE INDEX idx_sessions_user_date ON study_sessions(user_id, session_date);

-- Polymorphic tag lookup
CREATE INDEX idx_entity_tags_type_id ON entity_tags(entity_type, entity_id);
```

---

## 5. API Design

### 5.1 API Conventions

- **Versioning:** Path-based (`/api/v1/`)
- **Authentication:** JWT Bearer token in `Authorization` header
- **Updates:** PATCH only (no PUT) — partial updates, send only changed fields
- **Pagination:** `?page=0&size=20&sort=createdAt,desc`
- **Error Format:** RFC 7807 Problem Details

### 5.2 Response Envelope

```json
// Successful list response
{
  "data": [...],
  "meta": { "page": 0, "size": 20, "totalElements": 47, "totalPages": 3 }
}

// Successful single resource
{
  "data": { "id": 1, "name": "DSA", ... }
}

// Error response (RFC 7807)
{
  "type": "VALIDATION_ERROR",
  "title": "Validation Failed",
  "status": 400,
  "detail": "One or more fields have validation errors",
  "errors": [{ "field": "title", "message": "must not be blank" }]
}
```

### 5.3 Endpoint Summary

```
Authentication (Public)
  POST /api/v1/auth/register     → Create account, return JWT
  POST /api/v1/auth/login        → Authenticate, return JWT
  POST /api/v1/auth/refresh      → Exchange refresh token

Categories (Protected)
  GET    /api/v1/categories      → List with computed progress
  POST   /api/v1/categories      → Create
  PATCH  /api/v1/categories/:id  → Partial update

Topics (Protected)
  GET    /api/v1/topics?categoryId=&page=&size= → Paginated tree
  GET    /api/v1/topics/:id      → Single with children + progress
  POST   /api/v1/topics          → Create (validates depth ≤ 2)
  PATCH  /api/v1/topics/:id      → Update (validates status transition)
  DELETE /api/v1/topics/:id      → Delete

Todos (Protected)
  GET    /api/v1/todos?date=&status=&priority=&page=&size= → Filtered list
  GET    /api/v1/todos/today     → Today's tasks (convenience)
  POST   /api/v1/todos           → Create
  PATCH  /api/v1/todos/:id       → Partial update
  PATCH  /api/v1/todos/:id/complete → Mark complete (sets completed_at)
  DELETE /api/v1/todos/:id       → Delete

Dashboard (Protected — 3 parallel endpoints)
  GET /api/v1/dashboard/today-tasks       → {pending, completed, total}
  GET /api/v1/dashboard/category-progress → [{categoryName, progress%}]
  GET /api/v1/dashboard/weekly-summary    → {totalMinutes, dailyBreakdown}
```

---

## 6. Security Architecture

### 6.1 Authentication Flow

```
┌──────────┐                    ┌──────────────┐              ┌──────────┐
│  Client  │                    │ Spring Boot  │              │PostgreSQL│
│ (React)  │                    │  Backend     │              │          │
└────┬─────┘                    └──────┬───────┘              └────┬─────┘
     │                                 │                           │
     │ POST /auth/login                │                           │
     │ {username, password}            │                           │
     │────────────────────────────────►│                           │
     │                                 │ SELECT * FROM users       │
     │                                 │ WHERE username = ?        │
     │                                 │──────────────────────────►│
     │                                 │◄──────────────────────────│
     │                                 │                           │
     │                                 │ BCrypt.verify(password)   │
     │                                 │ JwtTokenProvider.generate │
     │                                 │                           │
     │  {accessToken, refreshToken}    │                           │
     │◄────────────────────────────────│                           │
     │                                 │                           │
     │ GET /api/v1/categories          │                           │
     │ Authorization: Bearer <JWT>     │                           │
     │────────────────────────────────►│                           │
     │                                 │ JwtAuthFilter:            │
     │                                 │  1. Extract token         │
     │                                 │  2. Validate signature    │
     │                                 │  3. Check expiry          │
     │                                 │  4. Set SecurityContext   │
     │                                 │                           │
     │                                 │ Controller:               │
     │                                 │  userId = getCurrentUser()│
     │                                 │──────────────────────────►│
     │                                 │  SELECT ... WHERE         │
     │                                 │  user_id = ?              │
     │                                 │◄──────────────────────────│
     │  {data: [...categories]}        │                           │
     │◄────────────────────────────────│                           │
```

### 6.2 Security Measures

| Measure | Implementation |
|---------|---------------|
| Password Storage | BCrypt (10 rounds) |
| Token Algorithm | HS384 (HMAC-SHA384) |
| Access Token TTL | 1 hour |
| Refresh Token TTL | 7 days |
| Session | Stateless (no server-side sessions) |
| CORS | Whitelisted origins only |
| Data Isolation | All queries scoped by `user_id` from JWT |
| Input Validation | Bean Validation (server) + Zod (client) |
| SQL Injection | Parameterized queries via JPA |
| Optimistic Locking | `@Version` field prevents concurrent overwrites |

---

## 7. Design Patterns & Principles

### 7.1 SOLID Principles Applied

| Principle | Application |
|-----------|------------|
| **S** — Single Responsibility | Each service handles one domain: TopicTreeService for tree ops, DashboardService for aggregation, TodoService for task management |
| **O** — Open/Closed | `TodoSpecification` — add new filters without modifying existing code; Tag system for extensible categorization |
| **L** — Liskov Substitution | Service interfaces (`TopicService`, `TodoService`) allow swapping implementations (e.g., `SimpleInsightProvider` → `LlmInsightProvider` in future) |
| **I** — Interface Segregation | Separate service interfaces for read and write concerns; DTOs split into Request and Response |
| **D** — Dependency Inversion | Controllers depend on service interfaces, not implementations; Repository abstraction over raw SQL |

### 7.2 Additional Patterns

| Pattern | Where Used |
|---------|-----------|
| **Repository Pattern** | Spring Data JPA repositories abstract database access |
| **Specification Pattern** | `TodoSpecification` — dynamic query composition |
| **DTO Pattern** | Request/Response DTOs prevent entity leakage across layers |
| **Builder Pattern** | Lombok `@Builder` on all DTOs and entities |
| **Template Method** | BaseEntity with auditing hooks (createdAt, updatedAt) |
| **Strategy Pattern** | Future-ready: service interfaces for swappable implementations |
| **Observer Pattern** | React Query's cache invalidation on mutations |

---

## 8. Deployment Architecture

### 8.1 Local Development

```
┌─────────────────────────────────────────────────────┐
│                  Developer Machine                   │
│                                                     │
│  ┌──────────────┐    ┌──────────────────────────┐   │
│  │ Vite Dev     │    │ Spring Boot              │   │
│  │ Server       │───►│ (embedded Tomcat)         │   │
│  │ :5173        │    │ :8080                     │   │
│  │              │    │                           │   │
│  │ React HMR    │    │ JAVA_HOME=JDK17          │   │
│  └──────────────┘    └────────────┬──────────────┘   │
│        │ proxy /api                │ JDBC             │
│        └──────────────┐            │                  │
│                       ▼            ▼                  │
│                  ┌──────────────────────┐             │
│                  │  Docker Compose      │             │
│                  │  ┌────────────────┐  │             │
│                  │  │ PostgreSQL 15  │  │             │
│                  │  │ :5434          │  │             │
│                  │  └────────────────┘  │             │
│                  └──────────────────────┘             │
└─────────────────────────────────────────────────────┘
```

### 8.2 Production Architecture (Recommended)

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌────────────┐
│          │     │              │     │              │     │            │
│  CDN     │────►│ Vercel /     │     │ Render /     │────►│ Managed    │
│(Cloudflr)│     │ Netlify      │     │ Railway /    │     │ PostgreSQL │
│          │     │              │     │ AWS ECS      │     │ (Supabase/ │
│          │     │ Static React │     │              │     │  Neon)     │
│          │     │ Build        │     │ Spring Boot  │     │            │
│          │     │              │     │ Container    │     │            │
└──────────┘     └──────────────┘     └──────────────┘     └────────────┘
                        │                    │
                        │  /api/* proxy      │
                        └────────────────────┘
```

---

## 9. Data Flow Diagrams

### 9.1 User Registration Flow

```
User → [Register Form] → POST /auth/register
                              │
                              ▼
                     [Validate DTO] ──error──► 400 Validation Error
                              │
                              ▼
                     [Check username/email unique] ──exists──► 409 Conflict
                              │
                              ▼
                     [BCrypt hash password]
                              │
                              ▼
                     [Save User to DB]
                              │
                              ▼
                     [Generate JWT pair]
                              │
                              ▼
                     Return {accessToken, refreshToken, user}
```

### 9.2 Topic Status Transition

```
                  ┌─────────────┐
                  │ NOT_STARTED │
                  └──────┬──────┘
                         │
                    ┌────▼────┐
              ┌─────│IN_PROGR.│─────┐
              │     └─────────┘     │
              │                     │
              ▼                     ▼
        ┌───────────┐        ┌───────────┐
        │NOT_STARTED│        │ COMPLETED │
        └───────────┘        └─────┬─────┘
                                   │
                              ┌────▼────┐
                              │IN_PROGR.│
                              └─────────┘

  Validated server-side: Map<CurrentStatus, Set<AllowedNextStatus>>
  NOT_STARTED → {IN_PROGRESS}
  IN_PROGRESS → {NOT_STARTED, COMPLETED}
  COMPLETED   → {IN_PROGRESS}
```

### 9.3 Dashboard Data Loading (Parallel)

```
DashboardPage mounts
       │
       ├──► useQuery(dashboardKeys.todayTasks)  ──► GET /dashboard/today-tasks
       │                                              └──► {pending:5, completed:3, total:8}
       │
       ├──► useQuery(dashboardKeys.categoryProgress) ──► GET /dashboard/category-progress
       │                                                   └──► [{name:"DSA", progress:45%}, ...]
       │
       └──► useQuery(dashboardKeys.weeklySummary)   ──► GET /dashboard/weekly-summary
                                                         └──► {totalMinutes:480, daily:{...}}

  All 3 queries fire simultaneously via React Query
  Each shows independent loading skeleton
  No waterfall — total load time = max(individual times)
```

---

## 10. Scalability Considerations

### 10.1 Current Capacity (Single Instance)

| Metric | Capacity |
|--------|----------|
| Concurrent Users | ~100-500 (single Tomcat thread pool) |
| Database Size | Up to 100K rows per table without issues |
| Response Time (p95) | <100ms for CRUD, <200ms for aggregations |

### 10.2 Scaling Path

| Phase | Scale Approach |
|-------|---------------|
| **Phase 1** (Current) | Single instance, vertical scaling |
| **Phase 2** | Read replicas for dashboard queries |
| **Phase 3** | Horizontal backend scaling (stateless JWT enables this) |
| **Phase 4** | Redis cache for frequently-accessed dashboard data |
| **Phase 5** | CDN for frontend, API Gateway for rate limiting |

### 10.3 Performance Optimizations

- **No denormalized fields:** Progress computed on-the-fly (trades CPU for consistency)
- **Split dashboard endpoints:** 3 lightweight queries instead of 1 heavy query
- **Paginated responses:** Prevents loading entire dataset
- **Query key factories:** Precise React Query cache invalidation
- **Optimistic updates:** UI updates instantly, syncs in background
- **Database indexes:** On all foreign keys and common filter columns

---

## 11. Future Extensibility

| Phase | Feature | Architecture Impact |
|-------|---------|-------------------|
| Phase 2 | Question Bank | New entity + service, existing tag system reused |
| Phase 3 | Full Analytics | Recharts integration, new dashboard endpoints |
| Phase 4 | Learning Journal | Rich text (TipTap), entity_tags for tagging |
| Phase 5 | LLM Integration | New `StudyInsightProvider` interface (Strategy pattern swap) |
| Phase 6 | Git Integration | External API adapter, async processing |
| Phase 7 | Gamification | Badge entity + achievement rules engine |

---

## 12. Technology Stack Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                        TECHNOLOGY STACK                          │
├──────────────┬──────────────────────────────────────────────────┤
│              │ React 19, TypeScript (strict), Vite 8            │
│   Frontend   │ Tailwind CSS v4, shadcn-style components         │
│              │ TanStack React Query v5, React Router v7         │
│              │ Zod validation, React Hook Form, Lucide icons    │
├──────────────┼──────────────────────────────────────────────────┤
│              │ Java 17, Spring Boot 3.3.11                      │
│   Backend    │ Spring Security (JWT, BCrypt)                    │
│              │ Spring Data JPA, Hibernate 6.5, JPA Specs        │
│              │ Flyway 10.10, Lombok, MapStruct                  │
├──────────────┼──────────────────────────────────────────────────┤
│   Database   │ PostgreSQL 15 (CHECK constraints, indexes)       │
├──────────────┼──────────────────────────────────────────────────┤
│   DevOps     │ Docker Compose, Maven, npm                       │
├──────────────┼──────────────────────────────────────────────────┤
│   Quality    │ Cursor Rules (4 .mdc files), Excalidraw diagrams │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## Appendix A: Seed Categories

| Category | Icon | Example Topics |
|----------|------|---------------|
| DSA | code-2 | Arrays, Two Pointers, Sliding Window, Binary Search, Trees, Graphs, DP |
| Java Core | coffee | OOP, Collections, Strings, Generics, Exception Handling |
| Java Advanced | layers | Multithreading, CompletableFuture, JVM Memory, Streams |
| Spring Boot | leaf | IoC/DI, Bean Lifecycle, Spring Data JPA, Security, AOP |
| SQL | database | JOINs, Indexing, Query Optimization, Transactions, Window Functions |
| System Design | network | URL Shortener, Rate Limiter, Chat System, Payment System |
| Design Patterns | puzzle | Creational (4), Structural (3), Behavioral (4) |
| SOLID Principles | shield | S, O, L, I, D with Java examples |
| Behavioral | users | STAR Stories, Leadership, Conflict Resolution |

---

## Appendix B: Non-Functional Requirements

| Requirement | Target | Approach |
|-------------|--------|----------|
| Response Time | <200ms (p95) | Indexed queries, no N+1, split endpoints |
| Availability | 99.5% | Stateless backend, managed DB, health checks |
| Data Consistency | Strong | DB transactions, optimistic locking, validated transitions |
| Security | OWASP Top 10 | JWT, BCrypt, parameterized queries, CORS, input validation |
| Usability | Mobile-responsive | Tailwind responsive classes, Sheet for mobile nav |
| Accessibility | WCAG 2.1 AA | Semantic HTML, ARIA roles, keyboard navigation |
