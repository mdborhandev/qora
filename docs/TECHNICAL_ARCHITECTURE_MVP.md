# Qora — Technical Architecture Document (MVP Phase 1)

**Version:** 1.0  
**Date:** July 2026  
**Status:** Draft — Pre-Development  
**Scope:** MVP (Phase 1) modules only

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Tech Stack](#2-tech-stack)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Project Structure](#4-project-structure)
5. [Multi-Tenancy Architecture](#5-multi-tenancy-architecture)
6. [Database Architecture](#6-database-architecture)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [API Architecture](#8-api-architecture)
9. [Real-Time Architecture (SignalR)](#9-real-time-architecture-signalr)
10. [Queue & Token Engine](#10-queue--token-engine)
11. [Appointment & Scheduling Engine](#11-appointment--scheduling-engine)
12. [Notification System](#12-notification-system)
13. [Display & Digital Signage](#13-display--digital-signage)
14. [Background Job Processing](#14-background-job-processing)
15. [Caching Strategy](#15-caching-strategy)
16. [Localization Architecture](#16-localization-architecture)
17. [Security](#17-security)
18. [Deployment Architecture](#18-deployment-architecture)
19. [Development Workflow](#19-development-workflow)
20. [Non-Functional Requirements](#20-non-functional-requirements)

---

## 1. System Overview

Qora is a cloud-based, multi-tenant Digital Queue Management System (DQMS) targeting hospitals, diagnostic centers, banks, and government offices in Bangladesh. The MVP delivers the core queue lifecycle — from token generation through real-time display — with appointment scheduling, multi-channel notifications, and Bengali/English localization.

### Design Principles

- **Vertical-agnostic core** — The data model and queue engine work identically across hospitals, banks, and government offices. Vertical-specific language (patient, customer, citizen) is pushed into a configurable label/terminology layer.
- **Unified Visit Journey** — One engine handles the full lifecycle: arrival → token → queue → service → completion. Not four separate subsystems bolted together.
- **Event-sourced token state** — Every token state transition is recorded as an immutable event, enabling real-time sync, analytics, and audit without additional infrastructure.
- **Multi-tenant from day one** — Organization → Branch → Department → Counter hierarchy with data isolation at the database level.
- **Offline-resilient displays** — Display boards cache the last known state and recover automatically when connectivity resumes.

### MVP Module Scope

| Module | Priority | Complexity |
|--------|----------|------------|
| Identity & Authentication | P0 | Medium |
| Organization & Branch Management | P0 | Medium |
| Queue & Token Management | P0 | High |
| Display & Digital Signage | P0 | Medium |
| Appointment & Scheduling | P0 | High |
| Notification & Communication | P0 | Medium |
| Localization (Bengali/English) | P1 | Low |

---

## 2. Tech Stack

### Backend

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Runtime | ASP.NET Core 9 | High performance, strong typing, excellent PostgreSQL/SignalR ecosystem |
| ORM | Entity Framework Core 9 | Code-first migrations, LINQ queries, change tracking |
| API Style | REST (JSON) | Simplicity, broad client compatibility, easy third-party integration |
| Real-Time | SignalR | Native ASP.NET integration, WebSocket + fallback transports, built-in group management |
| Background Jobs | Hangfire | Persistent job storage, dashboard, retry logic, .NET native |
| Auth | JWT Bearer + Refresh Tokens | Stateless API auth, mobile-friendly, refresh rotation |
| Validation | FluentValidation | Composable, testable validation rules |

### Data Layer

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16 | JSONB for flexible fields, LISTEN/NOTIFY for events, strong ACID, proven at scale |
| Cache | Redis 7 | Session store, token position caching, rate limiting, pub/sub for cross-instance events |
| Search | PostgreSQL FTS | Full-text search for names, tokens — avoids external search engine for MVP |

### Frontend

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Framework | React 18+ | Component-based, large ecosystem, SignalR client support |
| Styling | Bootstrap 5 + React-Bootstrap | Mature component library, responsive grid, extensive pre-built components |
| State Management | Zustand | Lightweight, no boilerplate, works well with SignalR |
| Router | React Router v6 | Standard SPA routing |
| i18n | react-i18next | Key-value translation, namespace splitting, interpolation |
| HTTP Client | Axios | Interceptors for auth token refresh, request/response logging |
| Tables | Tabulator (react-tabulator) | Feature-rich data tables with sorting, filtering, pagination, export |
| Charts | Recharts | Lightweight, composable, React-native |
| Forms | React Hook Form + Zod | Minimal re-renders, schema-based validation |

### Infrastructure

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Hosting | Hetzner VPS (CX32/CX42) | Cost-effective, European DC, adequate for MVP scale |
| Container | Docker + Docker Compose | Consistent dev/prod parity, easy deployment |
| Reverse Proxy | Nginx | SSL termination, static asset serving, rate limiting |
| Process Manager | systemd / Docker restart | Auto-restart on crash |
| Monitoring | Uptime Kuma + Serilog | Self-hosted uptime checks, structured logging |
| CI/CD | GitHub Actions | Build → test → deploy pipeline |

### Notification Channels

| Channel | Provider (MVP) | Fallback |
|---------|----------------|----------|
| SMS | Twilio / local BD gateway | — |
| WhatsApp | WhatsApp Business API (Cloud) | SMS |
| Email | SMTP (self-hosted or SendGrid free) | — |

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │ Staff Web │  │ Admin    │  │ Display  │  │ Public Appointment│ │
│  │ Dashboard │  │ Portal   │  │ Boards   │  │ Booking Page     │ │
│  └─────┬─────┘  └─────┬────┘  └─────┬────┘  └────────┬─────────┘ │
│        │              │              │                │           │
└────────┼──────────────┼──────────────┼────────────────┼───────────┘
         │              │              │                │
         │         HTTPS + WSS (SignalR)                │
         │              │              │                │
┌────────┼──────────────┼──────────────┼────────────────┼───────────┐
│        ▼              ▼              ▼                ▼           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                     NGINX (Reverse Proxy)                    │ │
│  │              SSL Termination · Rate Limiting                  │ │
│  │              Static Assets · Load Balancing                  │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                           │                                      │
│  ┌────────────────────────▼────────────────────────────────────┐ │
│  │              ASP.NET Core 9 Web API                          │ │
│  │                                                               │ │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌─────────────┐ │ │
│  │  │ Auth &    │ │ Queue &   │ │ Appointment│ │ Notification│ │ │
│  │  │ Identity  │ │ Token     │ │ Scheduling │ │ Engine      │ │ │
│  │  │ Module    │ │ Engine    │ │ Engine     │ │             │ │ │
│  │  └───────────┘ └───────────┘ └───────────┘ └─────────────┘ │ │
│  │                                                               │ │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌─────────────┐ │ │
│  │  │ Org &     │ │ Display   │ │ Locali-   │ │ Background  │ │ │
│  │  │ Branch    │ │ Service   │ │ zation    │ │ Jobs        │ │ │
│  │  │ Module    │ │           │ │ Service   │ │ (Hangfire)  │ │ │
│  │  └───────────┘ └───────────┘ └───────────┘ └─────────────┘ │ │
│  │                                                               │ │
│  │  ┌────────────────────────────────────────────────────────┐  │ │
│  │  │              SignalR Hubs                                │  │ │
│  │  │  QueueHub  ·  DisplayHub  ·  NotificationHub           │  │ │
│  │  └────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────┬───────────────────────────────────┘ │
│                             │                                     │
│  ┌──────────────────────────▼───────────────────────────────────┐ │
│  │                      DATA LAYER                               │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │ │
│  │  │ PostgreSQL 16 │  │   Redis 7    │  │  File Storage    │   │ │
│  │  │ (Primary DB)  │  │  (Cache/Pub) │  │  (Local/S3)      │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                  EXTERNAL SERVICES                           │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │ │
│  │  │ WhatsApp  │  │ SMS      │  │ Email    │  │ bKash/Nagad│  │ │
│  │  │ Business  │  │ Gateway  │  │ SMTP     │  │ (V2)       │  │ │
│  │  │ API       │  │          │  │          │  │            │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### Request Flow — Token Creation

```
1. Staff clicks "Issue Token" on dashboard
2. POST /api/v1/organizations/{orgId}/branches/{branchId}/tokens
3. API validates request (auth, permissions, branch access)
4. Queue Engine assigns token number, priority, estimated wait
5. Token event persisted (event-sourced)
6. Redis cache updated (position, queue state)
7. SignalR broadcasts to DisplayHub group (branch-specific)
8. Display boards update in real-time
9. Notification queued: SMS/WhatsApp to customer ("Your token is T-042")
10. Hangfire processes notification delivery async
```

---

## 4. Project Structure

### Solution Layout

```
qora/
├── src/
│   ├── Qora.Api/                          # ASP.NET Core Web API host
│   │   ├── Controllers/                   # API controllers (by module)
│   │   │   ├── AuthController.cs
│   │   │   ├── OrganizationsController.cs
│   │   │   ├── BranchesController.cs
│   │   │   ├── QueuesController.cs
│   │   │   ├── TokensController.cs
│   │   │   ├── AppointmentsController.cs
│   │   │   ├── DisplayController.cs
│   │   │   └── NotificationsController.cs
│   │   ├── Hubs/                           # SignalR hubs
│   │   │   ├── QueueHub.cs
│   │   │   ├── DisplayHub.cs
│   │   │   └── NotificationHub.cs
│   │   ├── Middleware/                     # Request pipeline
│   │   │   ├── TenantMiddleware.cs         # Resolves tenant from header/token
│   │   │   ├── RequestLoggingMiddleware.cs
│   │   │   └── ExceptionHandlingMiddleware.cs
│   │   ├── Filters/                        # Action filters
│   │   │   └── ValidateTenantAccessFilter.cs
│   │   ├── Program.cs                      # Entry point, DI registration
│   │   └── appsettings.json
│   │
│   ├── Qora.Core/                          # Domain layer (no dependencies)
│   │   ├── Entities/                       # Domain entities
│   │   │   ├── Organization.cs
│   │   │   ├── Branch.cs
│   │   │   ├── Department.cs
│   │   │   ├── Service.cs
│   │   │   ├── Counter.cs
│   │   │   ├── WorkingHours.cs
│   │   │   ├── User.cs
│   │   │   ├── Role.cs
│   │   │   ├── Permission.cs
│   │   │   ├── Queue.cs
│   │   │   ├── Token.cs
│   │   │   ├── TokenEvent.cs              # Event-sourced state transitions
│   │   │   ├── Appointment.cs
│   │   │   ├── AppointmentSlot.cs
│   │   │   ├── Notification.cs
│   │   │   └── LocalizationEntry.cs
│   │   ├── Enums/                          # Domain enumerations
│   │   │   ├── TokenStatus.cs
│   │   │   ├── TokenPriority.cs
│   │   │   ├── QueueType.cs
│   │   │   ├── AppointmentStatus.cs
│   │   │   ├── NotificationChannel.cs
│   │   │   ├── NotificationStatus.cs
│   │   │   └── UserRole.cs
│   │   ├── Interfaces/                     # Domain contracts
│   │   │   ├── IQueueEngine.cs
│   │   │   ├── INotificationService.cs
│   │   │   ├── ITokenNumberGenerator.cs
│   │   │   └── ILocalizationProvider.cs
│   │   ├── Events/                         # Domain events
│   │   │   ├── TokenCreatedEvent.cs
│   │   │   ├── TokenCalledEvent.cs
│   │   │   ├── TokenTransferredEvent.cs
│   │   │   ├── TokenCompletedEvent.cs
│   │   │   └── TokenSkippedEvent.cs
│   │   └── Exceptions/                     # Domain exceptions
│   │       ├── QueueNotFoundException.cs
│   │       ├── TokenNotFoundException.cs
│   │       └── TenantAccessDeniedException.cs
│   │
│   ├── Qora.Application/                   # Application/service layer
│   │   ├── Services/                       # Business logic
│   │   │   ├── AuthService.cs
│   │   │   ├── OrganizationService.cs
│   │   │   ├── BranchService.cs
│   │   │   ├── QueueService.cs
│   │   │   ├── TokenService.cs
│   │   │   ├── AppointmentService.cs
│   │   │   ├── DisplayService.cs
│   │   │   ├── NotificationService.cs
│   │   │   └── LocalizationService.cs
│   │   ├── DTOs/                           # Data transfer objects
│   │   │   ├── Requests/
│   │   │   └── Responses/
│   │   ├── Mapping/                        # AutoMapper profiles
│   │   ├── Validators/                     # FluentValidation validators
│   │   └── Interfaces/                     # Application-layer contracts
│   │
│   ├── Qora.Infrastructure/                # Infrastructure layer
│   │   ├── Data/                           # EF Core configuration
│   │   │   ├── QoraDbContext.cs
│   │   │   ├── Configurations/             # Entity type configurations
│   │   │   ├── Migrations/                 # EF Core migrations
│   │   │   └── Seeds/                      # Seed data
│   │   │       └── PermissionSeed.cs
│   │   ├── Services/                       # Infrastructure implementations
│   │   │   ├── JwtTokenService.cs
│   │   │   ├── RedisCacheService.cs
│   │   │   ├── SmsNotificationProvider.cs
│   │   │   ├── WhatsAppNotificationProvider.cs
│   │   │   ├── EmailNotificationProvider.cs
│   │   │   └── FileStorageService.cs
│   │   ├── Repositories/                   # Repository implementations
│   │   ├── BackgroundJobs/                 # Hangfire jobs
│   │   │   ├── NotificationDeliveryJob.cs
│   │   │   ├── AppointmentReminderJob.cs
│   │   │   ├── QueueCleanupJob.cs
│   │   │   └── ReportGenerationJob.cs
│   │   └── Extensions/                    # DI registration extensions
│   │
│   └── Qora.Shared/                        # Shared kernel
│       ├── Results/                        # Result<T> pattern
│       ├── Pagination/                     # Pagination models
│       ├── Constants/                      # System constants
│       └── Extensions/                     # Utility extensions
│
├── tests/
│   ├── Qora.Core.Tests/
│   ├── Qora.Application.Tests/
│   ├── Qora.Infrastructure.Tests/
│   └── Qora.Api.Tests/
│
├── frontend/
│   ├── src/
│   │   ├── components/                     # Reusable UI components
│   │   │   ├── ui/                         # Base UI components (Button, Card, Modal)
│   │   │   ├── queue/                      # Queue-specific components
│   │   │   ├── appointment/                # Appointment components
│   │   │   ├── display/                    # Display board components
│   │   │   └── layout/                     # Layout components (Sidebar, Header)
│   │   ├── pages/                          # Route pages
│   │   │   ├── auth/                       # Login, Register
│   │   │   ├── dashboard/                  # Staff dashboard
│   │   │   ├── admin/                      # Organization/Branch admin
│   │   │   ├── queue/                      # Queue management views
│   │   │   ├── appointment/                # Appointment views
│   │   │   ├── display/                    # Display board views
│   │   │   └── public/                     # Public booking page
│   │   ├── hooks/                          # Custom React hooks
│   │   │   ├── useSignalR.ts
│   │   │   ├── useQueue.ts
│   │   │   └── useAuth.ts
│   │   ├── stores/                         # Zustand stores
│   │   ├── services/                       # API client services
│   │   ├── i18n/                           # Internationalization
│   │   │   ├── en.json
│   │   │   └── bn.json
│   │   ├── types/                          # TypeScript types
│   │   ├── utils/                          # Utility functions
│   │   └── App.tsx
│   ├── public/
│   ├── vite.config.ts
│   └── package.json
│
├── docker/
│   ├── Dockerfile.api                      # Backend Dockerfile
│   ├── Dockerfile.web                      # Frontend Dockerfile
│   └── docker-compose.yml                  # Full stack compose
│
├── docs/
│   ├── ROADMAP.md
│   ├── TECHNICAL_ARCHITECTURE_MVP.md       # This document
│   └── archive/
│
├── .github/
│   └── workflows/
│       ├── ci.yml                          # Build + test
│       └── deploy.yml                      # Deploy to VPS
│
└── scripts/
    ├── db-migrate.sh                       # Run EF migrations
    ├── db-seed.sh                          # Seed initial data
    └── deploy.sh                           # Deploy script
```

### Layer Dependency Rules

```
Qora.Api → Qora.Application → Qora.Core
                ↓
         Qora.Infrastructure → Qora.Core
```

- **Qora.Core** has zero external dependencies. Pure domain.
- **Qora.Application** depends only on Core interfaces. No EF, no HTTP.
- **Qora.Infrastructure** implements Core interfaces. Owns EF, Redis, external APIs.
- **Qora.Api** composes everything. Wires DI, configures middleware.

---

## 5. Multi-Tenancy Architecture

### Tenant Model

```
Super Admin (Qora platform)
 └── Organization (Tenant) ── owns billing, branding, settings
      └── Branch ── physical location, working hours, counters
           └── Department ── service grouping
                ├── Service ── bookable/queueable unit
                └── Counter ── physical service point
```

### Tenant Resolution

Every authenticated request carries a JWT containing the user's organization and branch claims. The `TenantMiddleware` extracts this and sets a `TenantContext` in DI scope.

```csharp
// TenantContext — scoped to each request
public class TenantContext
{
    public Guid OrganizationId { get; set; }
    public Guid BranchId { get; set; }
    public Guid UserId { get; set; }
    public string Role { get; set; }
}
```

### Data Isolation

**Strategy: Shared database, shared schema, OrganizationId column.**

Every tenant-scoped entity includes `OrganizationId` and `BranchId` as required columns. EF Core global filters automatically append `WHERE OrganizationId = @tenantId` to every query.

```csharp
// EF Core Global Filter — applied in OnModelCreating
modelBuilder.Entity<Token>().HasQueryFilter(
    t => t.OrganizationId == _tenantContext.OrganizationId);
```

**Row-Level Security (optional hardening):** PostgreSQL RLS policies as a safety net:

```sql
ALTER TABLE tokens ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tokens
    USING (organization_id = current_setting('app.current_tenant')::uuid);
```

### Cross-Tenant Operations

Super admin endpoints bypass tenant scoping via `[Authorize(Roles = "SuperAdmin")]` and a `BypassTenantFilter()` extension on DbContext.

---

## 6. Database Architecture

### Core Entity Relationships

```
Organization (1) ──< Branch (many)
Branch (1) ──< Department (many)
Department (1) ──< Service (many)
Branch (1) ──< Counter (many)
Counter (1) ──< WorkingHours (many, 7 records per branch)

Organization (1) ──< User (many)
User (many) ──< Role (many) [through UserRole]
Role (many) ──< Permission (many) [through RolePermission]

Branch (1) ──< Queue (many)
Service (1) ──< Queue (many)
Queue (1) ──< Token (many)
Token (1) ──< TokenEvent (many) [event-sourced history]
Counter (1) ──< Token (many) [currently serving]

Branch (1) ──< Appointment (many)
Service (1) ──< Appointment (many)
User (1) ──< Appointment (many) [staff assignment]
User (1) ──< AppointmentSlot (many) [staff availability]
```

### Key Tables

#### Tenancy & Auth

```sql
-- Organizations (Tenants)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    logo_url        VARCHAR(500),
    primary_color   VARCHAR(7) DEFAULT '#2563EB',
    settings        JSONB DEFAULT '{}',
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Branches
CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(200) NOT NULL,
    address         TEXT,
    phone           VARCHAR(20),
    latitude        DECIMAL(10, 8),
    longitude       DECIMAL(11, 8),
    timezone        VARCHAR(50) DEFAULT 'Asia/Dhaka',
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(organization_id, name)
);

-- Departments
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    name            VARCHAR(200) NOT NULL,
    display_order   INTEGER DEFAULT 0,
    is_active       BOOLEAN DEFAULT true,
    UNIQUE(branch_id, name)
);

-- Services
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    department_id   UUID REFERENCES departments(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    estimated_duration_minutes INTEGER DEFAULT 15,
    priority_weight INTEGER DEFAULT 0,
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(branch_id, name)
);

-- Counters
CREATE TABLE counters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    display_number  INTEGER NOT NULL,
    is_active       BOOLEAN DEFAULT true,
    UNIQUE(branch_id, name)
);

-- Counter-Service mapping (which counter serves which service)
CREATE TABLE counter_services (
    counter_id      UUID NOT NULL REFERENCES counters(id) ON DELETE CASCADE,
    service_id      UUID NOT NULL REFERENCES services(id) ON DELETE CASCADE,
    PRIMARY KEY (counter_id, service_id)
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(254) NOT NULL,
    password_hash   VARCHAR(200) NOT NULL,
    full_name       VARCHAR(200) NOT NULL,
    phone           VARCHAR(20),
    avatar_url      VARCHAR(500),
    is_active       BOOLEAN DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(organization_id, email)
);

-- Roles
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN DEFAULT false,
    UNIQUE(organization_id, name)
);

-- Permissions (static, seeded)
CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    module          VARCHAR(50) NOT NULL,
    action          VARCHAR(50) NOT NULL,
    description     TEXT
);

-- Role-Permission mapping
CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- User-Role mapping
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    branch_id       UUID REFERENCES branches(id), -- NULL = all branches
    PRIMARY KEY (user_id, role_id)
);
```

#### Queue & Token Engine

```sql
-- Queues (a queue instance per service per branch)
CREATE TABLE queues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    service_id      UUID NOT NULL REFERENCES services(id),
    name            VARCHAR(200) NOT NULL,
    queue_type      VARCHAR(20) NOT NULL DEFAULT 'walkin', -- walkin | virtual | priority
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active | paused | closed
    current_token   VARCHAR(50),
    max_capacity    INTEGER DEFAULT 200,
    created_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(branch_id, service_id)
);

-- Tokens
CREATE TABLE tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id),
    queue_id        UUID NOT NULL REFERENCES queues(id),
    service_id      UUID NOT NULL REFERENCES services(id),
    counter_id      UUID REFERENCES counters(id),
    token_number    VARCHAR(50) NOT NULL,
    display_number  INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'waiting',
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    queue_type      VARCHAR(20) NOT NULL DEFAULT 'walkin',
    customer_name   VARCHAR(200),
    customer_phone  VARCHAR(20),
    customer_meta   JSONB DEFAULT '{}',
    estimated_wait_minutes INTEGER,
    called_at       TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(queue_id, token_number)
);

CREATE INDEX idx_tokens_queue_status ON tokens(queue_id, status);
CREATE INDEX idx_tokens_branch_status ON tokens(branch_id, status);
CREATE INDEX idx_tokens_created ON tokens(created_at DESC);

-- Token Events (event-sourced state transitions)
CREATE TABLE token_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    token_id        UUID NOT NULL REFERENCES tokens(id),
    event_type      VARCHAR(30) NOT NULL,
    from_status     VARCHAR(20),
    to_status       VARCHAR(20) NOT NULL,
    counter_id      UUID,
    performed_by    UUID,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_token_events_token ON token_events(token_id, created_at);
```

#### Appointments

```sql
-- Appointment Slots (staff availability windows)
CREATE TABLE appointment_slots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id),
    service_id      UUID NOT NULL REFERENCES services(id),
    staff_user_id   UUID REFERENCES users(id),
    date            DATE NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    capacity        INTEGER DEFAULT 1,
    booked_count    INTEGER DEFAULT 0,
    is_available    BOOLEAN DEFAULT true,
    UNIQUE(branch_id, service_id, staff_user_id, date, start_time)
);

-- Appointments
CREATE TABLE appointments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id),
    service_id      UUID NOT NULL REFERENCES services(id),
    slot_id         UUID REFERENCES appointment_slots(id),
    staff_user_id   UUID REFERENCES users(id),
    token_id        UUID REFERENCES tokens(id),
    customer_name   VARCHAR(200) NOT NULL,
    customer_phone  VARCHAR(20) NOT NULL,
    customer_email  VARCHAR(254),
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    date            DATE NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    notes           TEXT,
    reminder_sent   BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_appointments_branch_date ON appointments(branch_id, date, status);
```

#### Notifications

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    recipient_phone VARCHAR(20),
    recipient_email VARCHAR(254),
    channel         VARCHAR(20) NOT NULL, -- sms | whatsapp | email
    template_key    VARCHAR(100) NOT NULL,
    variables       JSONB DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    external_id     VARCHAR(200),  -- provider message ID
    error_message   TEXT,
    retry_count     INTEGER DEFAULT 0,
    max_retries     INTEGER DEFAULT 3,
    scheduled_at    TIMESTAMPTZ DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_notifications_status ON notifications(status, scheduled_at)
    WHERE status IN ('pending', 'retrying');
```

#### Localization

```sql
CREATE TABLE localization_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    locale          VARCHAR(10) NOT NULL, -- en | bn
    key             VARCHAR(200) NOT NULL,
    value           TEXT NOT NULL,
    is_system       BOOLEAN DEFAULT false,
    UNIQUE(organization_id, locale, key)
);
```

### Token Number Generation Strategy

Token numbers follow the pattern `{ServicePrefix}-{Sequence}`:

| Service | Prefix | Example |
|---------|--------|---------|
| General | G | G-001, G-002 |
| Emergency | E | E-001 |
| Lab | L | L-001 |
| Prescription | P | P-001 |

Sequence resets daily. Managed via PostgreSQL sequence per service per day:

```sql
-- Daily sequence per service
CREATE SEQUENCE token_seq_{service_id}_{YYYYMMDD} START 1;
```

EF Core generates the sequence name dynamically. Fallback: Redis `INCR` with daily TTL.

---

## 7. Authentication & Authorization

### Authentication Flow

```
┌─────────┐     POST /api/v1/auth/login      ┌──────────┐
│  Client  │ ──────────────────────────────→  │   API    │
│          │ ←──────────────────────────────  │          │
│          │  { accessToken, refreshToken }   │          │
│          │                                   │          │
│          │  GET /api/v1/tokens              │          │
│          │  Authorization: Bearer {access}  │          │
│          │ ──────────────────────────────→  │          │
│          │ ←──────────────────────────────  │          │
└─────────┘     200 OK + data                └──────────┘
```

### JWT Token Structure

```json
{
  "sub": "user-uuid",
  "email": "admin@hospital.com",
  "org": "org-uuid",
  "branch": "branch-uuid",
  "roles": ["admin"],
  "permissions": ["queue.manage", "token.issue", "token.call"],
  "iat": 1721472000,
  "exp": 1721475600
}
```

### Token Lifecycle

| Token | Lifetime | Storage | Purpose |
|-------|----------|---------|---------|
| Access Token | 15 minutes | Memory (JS variable) | API authorization |
| Refresh Token | 7 days | HttpOnly secure cookie | Token renewal |
| Password Reset | 1 hour | Redis (one-time use) | Account recovery |

### Refresh Flow

```
1. Access token expires (401 response)
2. Client sends POST /api/v1/auth/refresh with refresh token
3. Server validates refresh token (not revoked, not expired)
4. Server issues new access + refresh token pair
5. Old refresh token revoked (rotation)
```

### Role-Based Access Control

**System Roles (seeded):**

| Role | Scope | Permissions |
|------|-------|-------------|
| SuperAdmin | Platform-wide | All permissions, cross-org access |
| OrgAdmin | Organization | Full org management |
| BranchManager | Branch | Branch operations, staff management |
| Staff | Assigned branch | Queue ops, token management, appointments |
| Display | Branch (read-only) | View queue status, display board data |

**Permission Codes:**

```
{module}.{action}

Examples:
  organization.manage    — CRUD organizations
  branch.manage          — CRUD branches
  queue.view             — View queue status
  queue.manage           — Create/pause/close queues
  token.issue            — Generate new tokens
  token.call             — Call next token
  token.transfer         — Transfer token to another counter
  token.cancel           — Cancel a token
  appointment.book       — Create appointments
  appointment.manage     — Reschedule/cancel appointments
  display.view           — Access display board data
  notification.send      — Trigger notifications
  settings.manage        — Organization/branch settings
```

### Authorization Flow

```csharp
[Authorize(Permissions.Token.Issue)]
[HttpPost]
public async Task<IActionResult> IssueToken(IssueTokenRequest request)
{
    // TenantContext automatically scoped to user's org/branch
    var result = await _tokenService.IssueToken(request);
    return Ok(result);
}
```

---

## 8. API Architecture

### Base URL & Versioning

```
https://api.qora.io/v1/
```

Versioning via URL path segment. Major versions only (no `/v1.1`).

### Endpoint Convention

```
/api/v1/organizations/{orgId}/branches/{branchId}/...
```

All endpoints are branch-scoped. Org-scoped endpoints omit the branch parameter.

### Core Endpoints (MVP)

#### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/auth/login` | Authenticate user |
| POST | `/api/v1/auth/refresh` | Refresh access token |
| POST | `/api/v1/auth/logout` | Revoke refresh token |
| POST | `/api/v1/auth/forgot-password` | Send reset email |
| POST | `/api/v1/auth/reset-password` | Reset password |

#### Organization & Branch

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/organizations` | List organizations (super admin) |
| POST | `/api/v1/organizations` | Create organization |
| GET | `/api/v1/organizations/{orgId}` | Get organization details |
| PUT | `/api/v1/organizations/{orgId}` | Update organization |
| GET | `/api/v1/organizations/{orgId}/branches` | List branches |
| POST | `/api/v1/organizations/{orgId}/branches` | Create branch |
| GET | `/api/v1/organizations/{orgId}/branches/{branchId}` | Get branch |
| PUT | `/api/v1/organizations/{orgId}/branches/{branchId}` | Update branch |
| GET | `/api/v1/branches/{branchId}/departments` | List departments |
| POST | `/api/v1/branches/{branchId}/departments` | Create department |
| GET | `/api/v1/branches/{branchId}/services` | List services |
| POST | `/api/v1/branches/{branchId}/services` | Create service |
| GET | `/api/v1/branches/{branchId}/counters` | List counters |
| POST | `/api/v1/branches/{branchId}/counters` | Create counter |

#### Queue & Token

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/branches/{branchId}/queues` | List active queues |
| POST | `/api/v1/branches/{branchId}/queues` | Create queue |
| PUT | `/api/v1/queues/{queueId}/status` | Pause/resume/close queue |
| POST | `/api/v1/queues/{queueId}/tokens` | Issue new token |
| GET | `/api/v1/queues/{queueId}/tokens` | List tokens in queue |
| POST | `/api/v1/tokens/{tokenId}/call` | Call next / specific token |
| POST | `/api/v1/tokens/{tokenId}/transfer` | Transfer to another counter |
| POST | `/api/v1/tokens/{tokenId}/complete` | Mark token as served |
| POST | `/api/v1/tokens/{tokenId}/skip` | Skip token (no-show) |
| POST | `/api/v1/tokens/{tokenId}/cancel` | Cancel token |
| GET | `/api/v1/branches/{branchId}/tokens/active` | Get all active tokens for branch |
| GET | `/api/v1/tokens/{tokenId}` | Get token details + event history |

#### Appointment

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/branches/{branchId}/appointments/slots` | Get available slots |
| POST | `/api/v1/branches/{branchId}/appointments` | Book appointment |
| GET | `/api/v1/branches/{branchId}/appointments` | List appointments (filtered) |
| GET | `/api/v1/appointments/{appointmentId}` | Get appointment details |
| PUT | `/api/v1/appointments/{appointmentId}` | Reschedule appointment |
| DELETE | `/api/v1/appointments/{appointmentId}` | Cancel appointment |
| POST | `/api/v1/appointments/{appointmentId}/checkin` | Check in (convert to token) |

#### Display

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/branches/{branchId}/display/now-serving` | Currently serving tokens |
| GET | `/api/v1/branches/{branchId}/display/waiting` | Waiting queue list |
| GET | `/api/v1/branches/{branchId}/display/stats` | Queue statistics |

#### User & Role Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/organizations/{orgId}/users` | List users |
| POST | `/api/v1/organizations/{orgId}/users` | Create user |
| PUT | `/api/v1/users/{userId}` | Update user |
| DELETE | `/api/v1/users/{userId}` | Deactivate user |
| GET | `/api/v1/organizations/{orgId}/roles` | List roles |
| POST | `/api/v1/organizations/{orgId}/roles` | Create role |
| PUT | `/api/v1/roles/{roleId}` | Update role permissions |

### Request/Response Envelope

```json
// Success
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 145
  }
}

// Error
{
  "success": false,
  "error": {
    "code": "TOKEN_QUEUE_FULL",
    "message": "Queue has reached maximum capacity",
    "details": { "queueId": "...", "currentCount": 200, "maxCapacity": 200 }
  }
}
```

### Rate Limiting

| Endpoint Category | Limit | Window |
|-------------------|-------|--------|
| Auth endpoints | 10 req | 1 minute |
| General API | 100 req | 1 minute |
| Display endpoints | 60 req | 1 minute |
| Webhook/callback | 30 req | 1 minute |

Implemented via Nginx `limit_req` for global, Redis sliding window for per-user.

---

## 9. Real-Time Architecture (SignalR)

### Hub Design

| Hub | Connection Groups | Purpose |
|-----|-------------------|---------|
| `QueueHub` | `branch:{branchId}` | Live queue updates to staff dashboards |
| `DisplayHub` | `display:{branchId}:{screenId}` | Display board content |
| `NotificationHub` | `user:{userId}` | Personal notifications to staff |

### SignalR Groups

Groups are branch-scoped. When a user connects, they join:

```
branch:{branchId}     — all staff in that branch
display:{branchId}    — all display boards in that branch
```

### Message Types

#### QueueHub Messages

```csharp
// Server → Client
await Clients.Group($"branch:{branchId}").SendAsync("TokenIssued", tokenDto);
await Clients.Group($"branch:{branchId}").SendAsync("TokenCalled", tokenDto);
await Clients.Group($"branch:{branchId}").SendAsync("TokenTransferred", tokenDto);
await Clients.Group($"branch:{branchId}").SendAsync("TokenCompleted", tokenDto);
await Clients.Group($"branch:{branchId}").SendAsync("QueueUpdated", queueStateDto);
await Clients.Group($"branch:{branchId}").SendAsync("CounterUpdated", counterStateDto);
```

#### DisplayHub Messages

```csharp
// Server → Display Boards
await Clients.Group($"display:{branchId}").SendAsync("NowServing", nowServingDto);
await Clients.Group($"display:{branchId}").SendAsync("WaitingQueue", waitingListDto);
await Clients.Group($"display:{branchId}").SendAsync("QueueStats", statsDto);
await Clients.Group($"display:{branchId}").SendAsync("AudioAnnounce", announcementDto);
```

### Client Connection

```typescript
// React hook for SignalR
const connection = new HubConnectionBuilder()
  .withUrl("/hubs/queue", { accessTokenFactory: () => getAccessToken() })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .configureLogging(LogLevel.Warning)
  .build();

connection.on("TokenIssued", (token) => {
  useQueueStore.getState().addToken(token);
});

connection.on("TokenCalled", (token) => {
  useQueueStore.getState().updateToken(token);
  playAudioAnnouncement(token);
});

connection.on("QueueUpdated", (state) => {
  useQueueStore.getState().syncState(state);
});

await connection.start();
await connection.invoke("JoinBranch", branchId);
```

### Reconnection Strategy

1. Automatic reconnect with exponential backoff (built into SignalR)
2. On reconnect: fetch full queue state via REST to resync
3. Display boards: cache last known state, show "Reconnecting..." banner
4. Heartbeat: server pings every 15s, client timeout at 30s

### Scale Considerations

For MVP single-server deployment, SignalR runs in-memory. For multi-instance scaling:

- Redis backplane: `AddStackExchangeRedisBackplane()` in SignalR config
- Redis Pub/Sub handles cross-instance message relay

---

## 10. Queue & Token Engine

### Queue Types

| Type | Description | Use Case |
|------|-------------|----------|
| Walk-in | Default queue, first-come-first-served | Hospital reception, bank lobby |
| Virtual | Pre-registered, remote join | Appointment patients, online booking |
| Priority | Weighted sorting by priority level | VIP, emergency, elderly |

### Token Lifecycle State Machine

```
                    ┌─────────────────┐
                    │                 │
        ┌───────────┤   CREATED       │
        │           │                 │
        │           └────────┬────────┘
        │                    │
        │                    ▼
        │           ┌─────────────────┐
        │           │                 │
        │           │    WAITING      │◄────────────┐
        │           │                 │             │
        │           └────────┬────────┘             │
        │                    │                      │
        │              Call Next                     │
        │                    │                      │
        │                    ▼                      │
        │           ┌─────────────────┐             │
        │           │                 │             │
        │           │    CALLED       │─────────────┤
        │           │                 │    Transfer  │
        │           └────────┬────────┘             │
        │                    │                      │
        │              Start Service                │
        │                    │                      │
        │                    ▼                      │
        │           ┌─────────────────┐             │
        │           │                 │             │
        │           │   IN_SERVICE    │─────────────┘
        │           │                 │
        │           └────────┬────────┘
        │                    │
        │              Complete
        │                    │
        │                    ▼
        │           ┌─────────────────┐
        │           │                 │
        └──────────►│   COMPLETED     │
         Cancel     │                 │
                    └─────────────────┘

     SKIPPED (no-show) ── triggered from CALLED or WAITING
```

### Token Status Values

| Status | Description |
|--------|-------------|
| `waiting` | In queue, waiting to be called |
| `called` | Called to a counter, waiting for staff to start |
| `in_service` | Staff actively serving |
| `completed` | Service finished |
| `skipped` | No-show / skipped by staff |
| `cancelled` | Cancelled by staff or customer |
| `transferred` | Moved to another counter (transient, transitions to waiting/called) |

### Queue Engine Algorithm

```
IssueToken(queueId, priority):
  1. Validate queue is active and not full
  2. Generate token number: {prefix}-{dailySequence}
  3. Calculate display_number = queue.lastDisplayNumber + 1
  4. Calculate estimated_wait = (position-1) * avgServiceDuration
  5. Create token record (status: waiting)
  6. Persist TokenCreatedEvent
  7. Update queue.current_token
  8. Update Redis cache (position map)
  9. Return token with position + estimated wait

CallNext(queueId, counterId):
  1. Get next waiting token (sorted by priority weight, then created_at)
  2. Validate counter serves this queue's service
  3. Update token: status → called, counter_id → counterId, called_at → now
  4. Persist TokenCalledEvent
  5. Update queue.current_token
  6. Recalculate estimated waits for remaining tokens
  7. Update Redis cache
  8. Broadcast via SignalR
  9. Trigger notification (optional: "Please proceed to Counter 3")
  10. Return token details

CompleteToken(tokenId):
  1. Validate token is in_service
  2. Update token: status → completed, completed_at → now
  3. Persist TokenCompletedEvent
  4. Release counter (mark available)
  5. Recalculate estimated waits
  6. Update Redis cache
  7. Broadcast via SignalR
  8. Log completion metrics

TransferToken(tokenId, toCounterId):
  1. Validate source and target counters serve same service
  2. Update token: counter_id → toCounterId
  3. Persist TokenTransferredEvent
  4. Broadcast via SignalR
  5. Return updated token
```

### Estimated Wait Time Calculation

```csharp
public int CalculateEstimatedWait(Queue queue, Token token)
{
    var position = GetTokenPosition(queue.Id, token.Id);
    var avgDuration = GetAverageServiceDuration(queue.ServiceId);
    var activeCounters = GetActiveCounterCount(queue.BranchId, queue.ServiceId);
    
    if (activeCounters == 0) return -1; // Unknown
    
    var batchesBefore = Math.Ceiling((double)position / activeCounters);
    return (int)(batchesBefore * avgDuration);
}
```

Average service duration is tracked per service via a rolling window of completed tokens (last 100 completions).

---

## 11. Appointment & Scheduling Engine

### Slot Generation

Slots are generated from branch working hours + service duration:

```csharp
// Example: Generate slots for a service with 15-min duration
// Branch open: 09:00-17:00, Slot capacity: 3 per slot
var slots = new List<AppointmentSlot>();
var time = workingHours.OpenTime;

while (time.Add(service.Duration) <= workingHours.CloseTime)
{
    slots.Add(new AppointmentSlot
    {
        Date = date,
        StartTime = time,
        EndTime = time.Add(service.Duration),
        Capacity = 3,
        StaffUserId = staffId
    });
    time = time.Add(service.Duration);
}
```

### Booking Flow

```
1. Customer selects service + date
2. API returns available slots (capacity > booked_count)
3. Customer selects time slot
4. API creates appointment + decrements slot.booked_count
5. Optional: converts to token automatically at check-in
6. Notification queued: confirmation + reminders
```

### Appointment → Token Conversion

When a customer checks in for an appointment:

```
POST /api/v1/appointments/{appointmentId}/checkin

1. Validate appointment exists and is scheduled
2. Create token with queue_type = 'virtual'
3. Link token_id to appointment
4. Update appointment status → checked_in
5. Token enters normal queue flow
```

### Reminder Schedule

| Timing | Channel | Content |
|--------|---------|---------|
| 24 hours before | WhatsApp + SMS | Appointment confirmation + details |
| 2 hours before | WhatsApp | "Your appointment is in 2 hours" |
| 15 minutes before | SMS | "Please arrive now" |

Handled by Hangfire recurring job checking every 15 minutes.

---

## 12. Notification System

### Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Queue/      │────→│ Notification │────→│   Redis      │
│  Appointment │     │ Queue        │     │   Queue      │
│  Engine      │     │ (In-Process) │     │              │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                          ┌───────▼────────┐
                                          │   Hangfire      │
                                          │   Worker        │
                                          │                 │
                                          │  ┌───────────┐  │
                                          │  │ Dispatch   │  │
                                          │  │ Router     │  │
                                          │  └─────┬─────┘  │
                                          │        │        │
                                          │  ┌─────▼─────┐  │
                                          │  │ Provider   │  │
                                          │  │ Registry   │  │
                                          │  └─────┬─────┘  │
                                          │        │        │
                                          └────────┼────────┘
                                                   │
                                    ┌──────────────┼──────────────┐
                                    │              │              │
                              ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
                              │ WhatsApp   │ │ SMS       │ │ Email     │
                              │ Provider   │ │ Provider  │ │ Provider  │
                              └───────────┘ └───────────┘ └───────────┘
```

### Notification Events

| Event | Template Key | Default Channels |
|-------|-------------|------------------|
| Token Issued | `token.issued` | SMS, WhatsApp |
| Token Called | `token.called` | SMS, WhatsApp |
| Token Near Turn | `token.near` | WhatsApp |
| Token Skipped | `token.skipped` | SMS, WhatsApp |
| Appointment Confirmed | `appointment.confirmed` | SMS, WhatsApp |
| Appointment Reminder | `appointment.reminder` | SMS, WhatsApp |
| Appointment Cancelled | `appointment.cancelled` | SMS, WhatsApp |

### Template Engine

Templates use Handlebars-style syntax with i18n support:

```json
{
  "token.issued": {
    "en": "Your token is {{tokenNumber}}. Estimated wait: {{estimatedWait}} minutes. Queue: {{serviceName}} at {{branchName}}.",
    "bn": "আপনার টোকেন হলো {{tokenNumber}}। আনুমানিক অপেক্ষা: {{estimatedWait}} মিনিট। সার্ভিস: {{serviceName}}, {{branchName}}।"
  }
}
```

### Channel Priority & Fallback

```csharp
public class NotificationRouter
{
    private readonly Dictionary<string, string[]> _fallbackChain = new()
    {
        ["whatsapp"] = new[] { "whatsapp", "sms" },
        ["sms"] = new[] { "sms" },
        ["email"] = new[] { "email" }
    };

    public async Task Dispatch(Notification notification)
    {
        var channels = _fallbackChain[notification.PreferredChannel];
        
        foreach (var channel in channels)
        {
            var provider = _providers[channel];
            var result = await provider.SendAsync(notification);
            
            if (result.Success) return;
            
            // Log failure, try next channel
            notification.RetryCount++;
            if (notification.RetryCount >= notification.MaxRetries)
                break;
        }
        
        // All channels failed — mark for manual review
        notification.Status = "failed";
    }
}
```

### WhatsApp Business API Integration

```csharp
public class WhatsAppNotificationProvider : INotificationProvider
{
    private readonly HttpClient _httpClient;
    
    public async Task<NotificationResult> SendAsync(Notification notification)
    {
        var payload = new
        {
            messaging_product = "whatsapp",
            to = notification.RecipientPhone,
            type = "template",
            template = new
            {
                name = notification.TemplateKey,
                language = new { code = notification.Locale == "bn" ? "bn" : "en" },
                components = BuildComponents(notification.Variables)
            }
        };
        
        var response = await _httpClient.PostAsJsonAsync(
            $"https://graph.facebook.com/v18.0/{_config.PhoneNumberId}/messages",
            payload);
        
        // Handle response, extract message ID
    }
}
```

---

## 13. Display & Digital Signage

### Display Board Types

| Board | Purpose | Refresh Rate |
|-------|---------|-------------|
| Now Serving | Shows tokens currently being served at each counter | Real-time (SignalR) |
| Waiting Queue | Lists upcoming tokens in order | Real-time (SignalR) |
| Summary Stats | Total waiting, avg wait, counters active | 30-second interval |
| Audio Call | Plays TTS announcement when token is called | Event-driven |

### Display Board Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  QORA — Dhaka Medical Center                    10:32 AM  │
│─────────────────────────────────────────────────────────────────│
│                                                                  │
│  NOW SERVING                                                     │
│  ┌──────────┬──────────┬──────────┬──────────┐                  │
│  │ Counter 1│ Counter 2│ Counter 3│ Counter 4│                  │
│  │  G-042   │  G-039   │  L-015   │  G-041   │                  │
│  │ General  │ General  │ Lab      │ General  │                  │
│  └──────────┴──────────┴──────────┴──────────┘                  │
│                                                                  │
│  NEXT IN QUEUE                                                    │
│  G-043  •  G-044  •  G-045  •  L-016  •  G-046                 │
│                                                                  │
│  Total Waiting: 23    Avg Wait: 12 min    Active Counters: 4    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Display Data Flow

```
1. Token state changes (call, complete, transfer)
2. QueueEngine persists event + updates Redis
3. SignalR broadcasts to DisplayHub group
4. Display board React app receives update
5. UI re-renders with animation (smooth transitions)
6. If audio enabled: TTS plays announcement
```

### Audio Announcements

Using Web Speech API (SpeechSynthesis) for text-to-speech:

```typescript
const announceToken = (token: TokenDto, counter: CounterDto) => {
  const text = currentLocale === 'bn'
    ? `টোকেন নম্বর ${token.displayNumber}, অনুগ্রহ করে কাউন্টার ${counter.displayNumber}-এ যান।`
    : `Token number ${token.displayNumber}, please proceed to counter ${counter.displayNumber}.`;
  
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = currentLocale === 'bn' ? 'bn-BD' : 'en-US';
  utterance.rate = 0.9;
  speechSynthesis.speak(utterance);
};
```

### Display Board Auto-Recovery

```typescript
// Display board connection manager
class DisplayBoardManager {
  private lastState: QueueState | null = null;
  
  async connect() {
    this.connection.onreconnecting(() => {
      this.showBanner('Reconnecting...');
      // Show cached state
      if (this.lastState) this.renderState(this.lastState);
    });
    
    this.connection.onreconnected(async () => {
      this.hideBanner();
      // Fetch full state to resync
      const state = await api.getQueueState(this.branchId);
      this.renderState(state);
    });
    
    this.connection.onclose(async () => {
      this.showBanner('Connection lost. Retrying...');
      // Auto-reconnect handled by SignalR
    });
  }
}
```

---

## 14. Background Job Processing

### Hangfire Configuration

```csharp
// Program.cs
builder.Services.AddHangfire(config => config
    .SetDataCompatibilityLevel(CompatibilityLevel.Version_180)
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UsePostgreSqlStorage(options =>
        options.UseNpgsqlConnection(builder.Configuration.GetConnectionString("Hangfire"))));

builder.Services.AddHangfireServer();
```

### MVP Job Definitions

| Job | Type | Schedule | Purpose |
|-----|------|----------|---------|
| `NotificationDeliveryJob` | Recurring | Every 1 minute | Process pending notifications |
| `AppointmentReminderJob` | Recurring | Every 15 minutes | Send upcoming appointment reminders |
| `QueueCleanupJob` | Recurring | Daily at 02:00 | Archive old tokens, reset daily sequences |
| `TokenWaitRecalcJob` | Recurring | Every 5 minutes | Recalculate estimated wait times |
| `QueueStatsJob` | Recurring | Every 5 minutes | Aggregate queue statistics |

### Job Implementation Example

```csharp
public class NotificationDeliveryJob
{
    private readonly INotificationRouter _router;
    private readonly QoraDbContext _db;

    [AutomaticRetry(Attempts = 3)]
    public async Task Execute()
    {
        var pending = await _db.Notifications
            .Where(n => n.Status == "pending" && n.ScheduledAt <= DateTime.UtcNow)
            .Take(50)
            .ToListAsync();

        foreach (var notification in pending)
        {
            try
            {
                notification.Status = "sending";
                await _db.SaveChangesAsync();
                
                await _router.Dispatch(notification);
                
                notification.Status = "sent";
                notification.SentAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                notification.RetryCount++;
                notification.ErrorMessage = ex.Message;
                if (notification.RetryCount >= notification.MaxRetries)
                    notification.Status = "failed";
            }
            
            await _db.SaveChangesAsync();
        }
    }
}
```

---

## 15. Caching Strategy

### Redis Key Schema

| Key Pattern | TTL | Purpose |
|-------------|-----|---------|
| `queue:{queueId}:positions` | 5 min | Token position map |
| `queue:{queueId}:state` | 5 min | Current queue state snapshot |
| `queue:{branchId}:summary` | 1 min | Branch-level queue summary |
| `token:{tokenId}:wait` | 5 min | Estimated wait for specific token |
| `branch:{branchId}:counters` | 5 min | Counter status map |
| `rate:{ip}:{endpoint}` | 1 min | Rate limiting counter |
| `auth:refresh:{tokenHash}` | 7 days | Refresh token store |
| `auth:reset:{tokenHash}` | 1 hour | Password reset token |

### Cache Invalidation

Write-through: API writes update Redis synchronously alongside PostgreSQL.

```
Token Created → DB INSERT → Redis SET queue:{id}:positions
Token Called  → DB UPDATE → Redis SET queue:{id}:positions, queue:{branchId}:summary
Token Complete → DB UPDATE → Redis SET queue:{id}:positions, DEL token:{id}:wait
```

### Cache-Aside for Read-Heavy Data

```csharp
public async Task<QueueStateDto> GetQueueState(Guid queueId)
{
    var cacheKey = $"queue:{queueId}:state";
    var cached = await _redis.GetStringAsync(cacheKey);
    
    if (cached != null)
        return JsonSerializer.Deserialize<QueueStateDto>(cached);
    
    var state = await LoadQueueStateFromDb(queueId);
    await _redis.SetStringAsync(cacheKey, JsonSerializer.Serialize(state),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1) });
    
    return state;
}
```

---

## 16. Localization Architecture

### Strategy

- **Two locales:** English (`en`) and Bengali (`bn`)
- **Dynamic labels:** Organization owners can customize field labels (e.g., "Patient" vs "Customer" vs "Citizen")
- **Database-backed:** `localization_entries` table per tenant
- **Frontend:** react-i18next with JSON resource bundles + dynamic overrides

### Label Customization (Vertical-Agnostic)

```json
{
  "labels": {
    "entity.primary": "Patient",       // Hospital
    "entity.primary": "Customer",      // Bank
    "entity.primary": "Citizen",       // Government
    "queue.title": "OPD Queue",        // Hospital
    "queue.title": "Service Queue",    // Bank
    "service.unit": "Consultation"     // Hospital
  }
}
```

Organization admins configure labels in settings. Frontend loads labels on auth and applies as overrides to base i18n keys.

### Bengali UI Considerations

- Bengali text is ~30% longer than English — layout must accommodate
- Use Bengali numerals option (১, ২, ৩) for display boards
- RTL not needed (Bengali is LTR)
- Font: Noto Sans Bengali (Google Fonts, CDN)

---

## 17. Security

### Authentication Security

- Passwords hashed with BCrypt (work factor 12)
- Refresh token rotation (each use invalidates old token)
- Rate limiting on auth endpoints (10 req/min per IP)
- Account lockout after 5 failed attempts (15-minute cooldown)

### API Security

- HTTPS enforced (HSTS header, Nginx redirect)
- CORS configured for frontend origin only
- Input validation on all endpoints (FluentValidation)
- SQL injection prevented by EF Core parameterized queries
- XSS prevented by React's automatic escaping + CSP headers
- Request size limits (1MB default, configurable)

### Data Security

- Sensitive fields encrypted at rest (phone numbers in notifications)
- Database connections encrypted (SSL mode: Require)
- Redis connections encrypted (TLS)
- API keys and secrets stored in environment variables, never in code
- Audit trail via token_events table

### Infrastructure Security

- SSH key-only authentication (no password login)
- Firewall: only ports 80, 443, 22 open
- Regular automated backups (pg_dump daily, Redis RDB)
- Nginx rate limiting and request throttling
- Docker containers run as non-root user

---

## 18. Deployment Architecture

### Production Setup

```
┌─────────────────────────────────────────────────┐
│              Hetzner VPS (CX32/CX42)             │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │              Docker Compose                   │ │
│  │                                               │ │
│  │  ┌───────────┐  ┌───────────┐  ┌─────────┐ │ │
│  │  │ Nginx     │  │ .NET API  │  │ React   │ │ │
│  │  │ :80/:443  │──│ :5000     │  │ :3000   │ │ │
│  │  │           │  │           │  │ (static)│ │ │
│  │  └───────────┘  └─────┬─────┘  └─────────┘ │ │
│  │                        │                     │ │
│  │  ┌───────────┐  ┌─────▼─────┐  ┌─────────┐ │ │
│  │  │ PostgreSQL│  │ Redis     │  │Hangfire │ │ │
│  │  │ :5432     │  │ :6379     │  │Dashboard│ │ │
│  │  └───────────┘  └───────────┘  │ :5001   │ │ │
│  │                                 └─────────┘ │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  /var/log/qora/     — Application logs       │ │
│  │  /var/backups/qora/ — Daily DB backups       │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Docker Compose (MVP)

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/ssl:/etc/nginx/ssl
    depends_on:
      - api
      - web

  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__Default=Host=postgres;Database=qora;Username=qora;Password=${DB_PASSWORD}
      - ConnectionStrings__Redis=redis:6379
      - Jwt__SecretKey=${JWT_SECRET}
      - Jwt__Issuer=qora
      - Jwt__Audience=qora
      - WhatsApp__AccessToken=${WHATSAPP_TOKEN}
      - WhatsApp__PhoneNumberId=${WHATSAPP_PHONE_ID}
      - Sms__ApiKey=${SMS_API_KEY}
    depends_on:
      - postgres
      - redis

  web:
    build:
      context: .
      dockerfile: docker/Dockerfile.web
    environment:
      - VITE_API_URL=https://api.qora.io
      - VITE_SIGNALR_URL=https://api.qora.io/hubs

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=qora
      - POSTGRES_USER=qora
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### Nginx Configuration

```nginx
upstream api {
    server api:5000;
}

server {
    listen 443 ssl http2;
    server_name api.qora.io;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # API
    location /api/ {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SignalR
    location /hubs/ {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=10r/m;

    location /api/v1/auth/ {
        limit_req zone=auth burst=5 nodelay;
    }

    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
}
```

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Restore & Build
        run: dotnet build --configuration Release
      
      - name: Test
        run: dotnet test --configuration Release --no-build
      
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/qora
            git pull origin main
            docker compose build
            docker compose up -d
            docker compose exec api dotnet ef database update
```

---

## 19. Development Workflow

### Local Development Setup

```bash
# 1. Clone repository
git clone https://github.com/your-org/qora.git
cd qora

# 2. Start infrastructure
docker compose -f docker-compose.dev.yml up -d postgres redis

# 3. Run database migrations
cd src/Qora.Api
dotnet ef database update

# 4. Seed initial data
dotnet run --seed

# 5. Start backend
dotnet run

# 6. Start frontend (new terminal)
cd frontend
npm install
npm run dev
```

### Branch Strategy

```
main          ← production, always deployable
  └── develop ← integration branch
       ├── feature/queue-engine
       ├── feature/appointment-scheduling
       ├── feature/display-board
       └── fix/token-number-overflow
```

### Sprint Cadence (MVP)

| Sprint | Duration | Focus |
|--------|----------|-------|
| Sprint 1 | 2 weeks | Auth, Organization, Branch, User Management |
| Sprint 2 | 2 weeks | Queue Engine, Token Management, Counter |
| Sprint 3 | 2 weeks | Display Board, Notifications, Real-Time |
| Sprint 4 | 2 weeks | Appointments, Localization, Dashboard |
| Testing | 1 week | Integration, load, UAT |
| Beta | 2 weeks | Deploy to 3 pilot locations |

---

## 20. Non-Functional Requirements

### Performance

| Metric | Target |
|--------|--------|
| API response time (p95) | < 200ms |
| SignalR message delivery | < 100ms |
| Display board update latency | < 500ms |
| Concurrent users per branch | 50+ |
| Token creation throughput | 100 tokens/minute |
| Display board connections | 200 per branch |

### Availability

| Metric | Target |
|--------|--------|
| Uptime | 99.5% (MVP) |
| Recovery Time Objective (RTO) | 1 hour |
| Recovery Point Objective (RPO) | 15 minutes (backup interval) |
| Max planned downtime | 2 hours/month |

### Scalability (MVP → V1)

| Dimension | MVP | V1 Target |
|-----------|-----|-----------|
| Organizations | 10 | 100 |
| Branches per org | 5 | 50 |
| Concurrent connections | 500 | 5,000 |
| Tokens per day | 10,000 | 100,000 |
| Database size | 10 GB | 100 GB |

### Monitoring

| Component | Tool | Metric |
|-----------|------|--------|
| Uptime | Uptime Kuma | HTTP health checks |
| Logs | Serilog → File + Seq | Structured application logs |
| Errors | Sentry (free tier) | Exception tracking |
| Performance | Application Insights / custom | Request duration, throughput |
| Database | pg_stat_statements | Query performance |
| Infrastructure | htop, docker stats | CPU, RAM, disk |

---

## Appendix A: Permission Matrix

| Module | Permission | SuperAdmin | OrgAdmin | BranchManager | Staff | Display |
|--------|-----------|------------|----------|---------------|-------|---------|
| Organization | manage | Yes | Yes | — | — | — |
| Branch | manage | Yes | Yes | Read | — | — |
| Department | manage | Yes | Yes | Yes | — | — |
| Service | manage | Yes | Yes | Yes | — | — |
| Counter | manage | Yes | Yes | Yes | — | — |
| User | manage | Yes | Yes | Read | — | — |
| Role | manage | Yes | Yes | — | — | — |
| Queue | view | Yes | Yes | Yes | Yes | Yes |
| Queue | manage | Yes | Yes | Yes | — | — |
| Token | issue | Yes | Yes | Yes | Yes | — |
| Token | call | Yes | Yes | Yes | Yes | — |
| Token | transfer | Yes | Yes | Yes | Yes | — |
| Token | cancel | Yes | Yes | Yes | Yes | — |
| Appointment | book | Yes | Yes | Yes | Yes | — |
| Appointment | manage | Yes | Yes | Yes | — | — |
| Display | view | Yes | Yes | Yes | Yes | Yes |
| Notification | send | Yes | Yes | Yes | — | — |
| Settings | manage | Yes | Yes | Branch | — | — |

---

## Appendix B: Database Migration Order

Migrations must be applied in this order due to foreign key dependencies:

1. `organizations`
2. `branches`, `roles`, `permissions`
3. `departments`, `users`, `role_permissions`, `user_roles`
4. `services`, `counters`, `counter_services`
5. `queues`, `token_events`
6. `tokens`
7. `appointment_slots`, `appointments`
8. `notifications`
9. `localization_entries`
10. Seed data (permissions, default roles, default admin)

---

## Appendix C: Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ASPNETCORE_ENVIRONMENT` | Yes | Development / Production |
| `ConnectionStrings__Default` | Yes | PostgreSQL connection string |
| `ConnectionStrings__Redis` | Yes | Redis connection string |
| `Jwt__SecretKey` | Yes | JWT signing key (min 32 chars) |
| `Jwt__Issuer` | Yes | JWT issuer |
| `Jwt__Audience` | Yes | JWT audience |
| `Jwt__AccessTokenExpiration` | No | Default 15 minutes |
| `Jwt__RefreshTokenExpiration` | No | Default 7 days |
| `WhatsApp__AccessToken` | Yes | WhatsApp Business API token |
| `WhatsApp__PhoneNumberId` | Yes | WhatsApp Business phone number ID |
| `WhatsApp__ApiVersion` | No | Default v18.0 |
| `Sms__ApiKey` | Yes | SMS gateway API key |
| `Sms__SenderId` | No | SMS sender ID |
| `FileStorage__Path` | No | Local file storage path |
| `Cors__Origins` | Yes | Allowed frontend origins |
| `Admin__DefaultPassword` | Yes | Default admin password (seed) |

---

*This document is a living reference. Update it as architectural decisions evolve during development.*
