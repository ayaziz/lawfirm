# LegalSaaS — Architecture & Product Design Document

**Version:** 1.0 — Final  
**Classification:** Internal — Implementation Ready  
**Date:** February 18, 2026  
**Author:** Technical Architecture & Product Design Team  
**Status:** APPROVED FOR IMPLEMENTATION

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Platform Overview](#2-platform-overview)
3. [Architecture — Logical & Deployment Views](#3-architecture--logical--deployment-views)
4. [Subdomain & Tenant Isolation Strategy](#4-subdomain--tenant-isolation-strategy)
5. [Data Model & Storage Architecture](#5-data-model--storage-architecture)
6. [Identity, Authentication & RBAC](#6-identity-authentication--rbac)
7. [API Layer Design](#7-api-layer-design)
8. [Azure Blob Storage Integration](#8-azure-blob-storage-integration)
9. [Environment Separation & DevOps](#9-environment-separation--devops)
10. [Security Controls](#10-security-controls)
11. [Observability, Monitoring & Logging](#11-observability-monitoring--logging)
12. [UX & Design Architecture](#12-ux--design-architecture)
13. [Public Website (V2)](#13-public-website-v2)
14. [Tenant Portal (MVP — Wave 1)](#14-tenant-portal-mvp--wave-1)
15. [Admin Portal (V2)](#15-admin-portal-v2)
16. [Delivery & Planning](#16-delivery--planning)
17. [Risk Assessment & Mitigation](#17-risk-assessment--mitigation)
18. [Appendices](#18-appendices)

---

## 1. Executive Summary

LegalSaaS is an enterprise-grade, multi-tenant SaaS platform purpose-built for law firms operating primarily in the MENA region. It delivers customer management, case management, document management, calendar, and firm administration capabilities through a secure, tenant-isolated architecture running on Microsoft Azure.

The platform comprises three components:

| Component | Scope | Target Release |
|---|---|---|
| **Public Website** | Marketing, pricing, subscription flow, platform info | V2 |
| **Tenant Portal** | Private tenant workspaces — full case/customer/document management | **MVP (Wave 1)** |
| **Admin Portal** | Internal SaaS administration — provisioning, billing, monitoring | V2 |

**Key Architectural Decisions (Finalized):**

- **Runtime:** .NET 8 (LTS) Web API + Azure SQL Database per-tenant
- **Frontend:** React 18 + TypeScript + Vite, deployed as static SPA
- **Identity:** Azure AD B2C for tenant users; Azure AD for Admin Portal operators
- **Storage:** Azure Blob Storage with tenant-isolated containers, SAS-token downloads
- **Hosting:** Azure App Service (API), Azure Static Web Apps / Azure CDN (SPA)
- **Database Strategy:** Database-per-tenant (logical isolation via Azure SQL Elastic Pools)
- **Tenant Routing:** Subdomain-based (`{tenant}.legalsaas.com`)
- **IaC:** Bicep (Azure Resource Manager)
- **CI/CD:** GitHub Actions

**Assumptions (Explicit):**

1. Target market is MENA-region law firms (5–200 users per firm)
2. Maximum 500 tenants in first 2 years; scaled to 5,000 tenants at maturity
3. All data residency within Azure region: UAE North (primary), West Europe (DR)
4. Arabic and English bilingual support required by V2; English-only for MVP
5. Platform team size: 3–5 engineers for Wave 1
6. Subscription billing is managed externally (Stripe or equivalent) with webhook integration
7. No on-premise deployment; SaaS-only
8. Legal document retention: minimum 10 years per tenant contract
9. SLA target: 99.9% uptime for Tenant Portal

---

## 2. Platform Overview

### 2.1 Component Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                        INTERNET / USERS                              │
└──────────────┬───────────────────┬───────────────────┬───────────────┘
               │                   │                   │
        ┌──────▼──────┐    ┌──────▼──────┐    ┌───────▼──────┐
        │   Public     │    │   Tenant    │    │    Admin     │
        │  Website     │    │   Portal    │    │   Portal     │
        │  (V2)        │    │  (MVP W1)   │    │   (V2)       │
        │              │    │             │    │              │
        │ legalsaas.com│    │ {t}.legal   │    │ admin.legal  │
        │              │    │  saas.com   │    │  saas.com    │
        └──────┬───────┘    └──────┬──────┘    └──────┬───────┘
               │                   │                   │
               └───────────┬───────┴───────────┬───────┘
                           │                   │
                    ┌──────▼──────┐     ┌──────▼──────┐
                    │   API       │     │  Admin API  │
                    │  Gateway    │     │  Gateway    │
                    │  (Tenant)   │     │  (Internal) │
                    └──────┬──────┘     └──────┬──────┘
                           │                   │
                    ┌──────▼──────────────────▼──────┐
                    │       Application Services       │
                    │  ┌─────┐ ┌─────┐ ┌──────────┐   │
                    │  │Tenant│ │Case │ │ Document │   │
                    │  │Mgmt  │ │Mgmt │ │ Mgmt     │   │
                    │  └──┬──┘ └──┬──┘ └────┬─────┘   │
                    │     │       │          │         │
                    └─────┼───────┼──────────┼─────────┘
                          │       │          │
               ┌──────────▼───────▼──┐  ┌───▼──────────┐
               │  Azure SQL Elastic  │  │ Azure Blob   │
               │  Pool (per-tenant   │  │ Storage      │
               │  databases)         │  │ (per-tenant  │
               └─────────────────────┘  │ containers)  │
                                        └──────────────┘
```

### 2.2 Technology Stack — Finalized

| Layer | Technology | Version | Rationale |
|---|---|---|---|
| **Frontend SPA** | React + TypeScript | 18.3 / 5.x | Component ecosystem, enterprise adoption |
| **Build Tool** | Vite | 5.x | Fast HMR, ESM-native |
| **UI Component Library** | Radix UI + Tailwind CSS | Latest | Accessible primitives, design token alignment |
| **State Management** | TanStack Query + Zustand | v5 / v4 | Server-state caching + minimal client state |
| **API Runtime** | .NET 8 Web API | 8.0 LTS | Performance, Azure-native, strong typing |
| **ORM** | Entity Framework Core | 8.x | Migrations, LINQ, multi-tenant DbContext |
| **Database** | Azure SQL Database | Latest | Relational, elastic pools, geo-replication |
| **Blob Storage** | Azure Blob Storage | v12 SDK | Document storage, SAS tokens, lifecycle |
| **Identity (Tenants)** | Azure AD B2C | Latest | Social + local auth, custom policies |
| **Identity (Admin)** | Microsoft Entra ID | Latest | Internal SSO, conditional access |
| **Caching** | Azure Cache for Redis | 6.x | Session, rate limiting, hot data |
| **Search** | Azure AI Search | Latest | Full-text across cases/documents (V2) |
| **Messaging** | Azure Service Bus | Standard | Async workflows, event-driven |
| **CDN** | Azure Front Door | Standard | Global edge, WAF, SSL termination |
| **IaC** | Bicep | Latest | Azure-native, modular deployment |
| **CI/CD** | GitHub Actions | N/A | Repo-integrated, matrix builds |
| **Monitoring** | Azure Monitor + App Insights | Latest | E2E tracing, alerting |

---

## 3. Architecture — Logical & Deployment Views

### 3.1 Logical Architecture

```
┌─────────────────────────── Presentation Layer ──────────────────────┐
│  Public Website SPA    │  Tenant Portal SPA    │  Admin Portal SPA  │
│  (React + Next.js SSR) │  (React + Vite)       │  (React + Vite)    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
┌────────────────────────── Edge / Gateway Layer ─────────────────────┐
│                    Azure Front Door (WAF + CDN)                      │
│                    ┌─────────────────────────┐                       │
│                    │  Subdomain Router        │                      │
│                    │  Rate Limiter             │                      │
│                    │  SSL/TLS Termination      │                      │
│                    └─────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
┌─────────────────────── Application Layer (.NET 8) ──────────────────┐
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ Tenant     │  │ Identity   │  │ Case       │  │ Document     │  │
│  │ Resolution │  │ & RBAC     │  │ Management │  │ Management   │  │
│  │ Middleware │  │ Middleware │  │ Service    │  │ Service      │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────────┘  │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ Customer   │  │ Calendar   │  │ Audit      │  │ Notification │  │
│  │ Management │  │ Service    │  │ Service    │  │ Service      │  │
│  │ Service    │  │            │  │            │  │              │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────────┘  │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                    │
│  │ User       │  │ Court      │  │ Backup     │                    │
│  │ Management │  │ Config     │  │ Service    │                    │
│  │ Service    │  │ Service    │  │            │                    │
│  └────────────┘  └────────────┘  └────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
┌─────────────────────── Infrastructure Layer ────────────────────────┐
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Azure SQL    │  │ Azure Blob   │  │ Azure Cache for Redis    │  │
│  │ Elastic Pool │  │ Storage      │  │                          │  │
│  │ (per-tenant  │  │ (per-tenant  │  │ (sessions, rate limits,  │
│  │  databases)  │  │  containers) │  │  tenant config cache)    │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Azure AD B2C │  │ Azure Key    │  │ Azure Service Bus        │  │
│  │              │  │ Vault        │  │ (async events)           │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐                                │
│  │ Azure Monitor│  │ App Insights │                                │
│  │ + Log        │  │ (APM)        │                                │
│  │ Analytics    │  │              │                                │
│  └──────────────┘  └──────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Deployment View (Azure)

| Resource | SKU / Tier | Instances | Purpose |
|---|---|---|---|
| **Azure Front Door** | Standard | 1 | Global load balancing, WAF, SSL, routing |
| **Azure App Service Plan** | P1v3 (Production) / B1 (Dev) | 2 (API + Admin API) | .NET 8 API hosting |
| **Azure Static Web Apps** | Standard | 3 | SPA hosting (Public, Tenant Portal, Admin) |
| **Azure SQL Elastic Pool** | Standard S3 (200 eDTU) → Premium P1 | 1 pool | Per-tenant databases |
| **Azure SQL — Platform DB** | S2 (50 DTU) | 1 | Tenant catalog, subscription metadata |
| **Azure Blob Storage** | StorageV2 (Hot + Cool) | 1 account | Per-tenant document containers |
| **Azure Cache for Redis** | C1 Standard | 1 | Caching, sessions |
| **Azure Key Vault** | Standard | 1 per environment | Secrets, certificates |
| **Azure AD B2C** | Standard | 1 tenant | Tenant user authentication |
| **Azure Service Bus** | Standard | 1 namespace | Async events (provisioning, notifications) |
| **Azure Monitor + App Insights** | Pay-as-you-go | 1 workspace | Logging, APM, alerting |
| **Azure AI Search** | Basic (V2) | 1 | Full-text search across cases/docs |

### 3.3 Network Topology

```
Internet
    │
    ▼
Azure Front Door (WAF Policy: OWASP 3.2, Rate Limiting)
    │
    ├── *.legalsaas.com → Azure Static Web Apps (Tenant SPA)
    ├── legalsaas.com → Azure Static Web Apps (Public Website)
    ├── admin.legalsaas.com → Azure Static Web Apps (Admin SPA)
    │
    ├── api.legalsaas.com → Azure App Service (Tenant API)
    │       │
    │       └── VNet Integrated → Private Endpoints
    │               ├── Azure SQL Elastic Pool
    │               ├── Azure Blob Storage
    │               ├── Azure Cache for Redis
    │               ├── Azure Key Vault
    │               └── Azure Service Bus
    │
    └── admin-api.legalsaas.com → Azure App Service (Admin API)
            │
            └── VNet Integrated → Private Endpoints (same VNet)
```

---

## 4. Subdomain & Tenant Isolation Strategy

### 4.1 Subdomain Strategy

| URL Pattern | Component | Access |
|---|---|---|
| `legalsaas.com` | Public Website | Public |
| `{tenant-slug}.legalsaas.com` | Tenant Portal SPA | Authenticated tenant users |
| `api.legalsaas.com` | Tenant API | Bearer token + tenant header |
| `admin.legalsaas.com` | Admin Portal SPA | Internal operators |
| `admin-api.legalsaas.com` | Admin API | Internal operators (Entra ID) |

**Tenant Slug Rules:**
- Lowercase alphanumeric + hyphens only
- 3–40 characters
- Unique across platform
- Immutable after provisioning
- Examples: `almansour`, `nile-law`, `cairo-legal-partners`

### 4.2 Tenant Resolution Flow

```
1. Browser → https://almansour.legalsaas.com
2. Azure Front Door → Static Web App (Tenant SPA bundle)
3. SPA boots → extracts subdomain "almansour" from window.location.host
4. SPA → GET api.legalsaas.com/health with Header: X-Tenant-Slug: almansour
5. API Tenant Resolution Middleware:
   a. Read X-Tenant-Slug header
   b. Lookup in Platform DB: tenants table → get tenant_id, db connection, status
   c. Cache in Redis (TTL: 5 min)
   d. Validate tenant status = Active
   e. Set TenantContext on HttpContext.Items
6. All downstream services use TenantContext.TenantId
7. EF Core DbContext resolves connection string from TenantContext
```

### 4.3 Isolation Model

| Resource | Isolation Level | Mechanism |
|---|---|---|
| **Database** | Database-per-tenant | Separate Azure SQL DB in Elastic Pool |
| **Blob Storage** | Container-per-tenant | Container: `tenant-{tenant_id}` |
| **Cache** | Key-prefix isolation | Redis key: `tenant:{tenant_id}:*` |
| **API** | Request-scoped context | `TenantContext` middleware on every request |
| **Auth Tokens** | Tenant-bound claims | JWT custom claim: `tenant_id` |
| **Audit Logs** | Tenant-scoped | `tenant_id` column in all audit tables |
| **Search Index** | Tenant-scoped filter | Security filter on `tenant_id` field |

### 4.4 Tenant Lifecycle

```
┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────────┐
│ Prospect  │───►│ Pending   │───►│ Active   │───►│ Suspended    │
│ (Website) │    │ Approval  │    │          │    │ (Non-payment)│
└──────────┘    └───────────┘    └──────────┘    └──────┬───────┘
                                       │                 │
                                       │           ┌─────▼───────┐
                                       │           │ Terminated  │
                                       └──────────►│ (Data       │
                                                   │  retained   │
                                                   │  90 days)   │
                                                   └─────────────┘
```

**Provisioning (via Admin Portal or automated):**
1. Create tenant record in Platform DB
2. Create Azure SQL Database in Elastic Pool
3. Run EF Core migrations on tenant database
4. Create Blob Storage container `tenant-{id}`
5. Create Azure AD B2C custom attribute for `tenantId`
6. Seed default data (roles, court templates for region)
7. Send onboarding email to Tenant Admin
8. Tenant Admin completes onboarding wizard (TA-08)

---

## 5. Data Model & Storage Architecture

### 5.1 Platform Database (Shared — `legalsaas-platform`)

This database stores tenant catalog and subscription metadata. It is NOT tenant-specific data.

```sql
-- Tenant catalog
CREATE TABLE tenants (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    slug                NVARCHAR(40) NOT NULL UNIQUE,
    display_name        NVARCHAR(200) NOT NULL,
    status              NVARCHAR(20) NOT NULL DEFAULT 'PendingApproval',
        -- PendingApproval | Active | Suspended | Terminated
    subscription_plan   NVARCHAR(50) NOT NULL,        -- Starter | Professional | Enterprise
    max_seats           INT NOT NULL DEFAULT 5,
    db_server           NVARCHAR(200) NOT NULL,
    db_name             NVARCHAR(100) NOT NULL,
    blob_container      NVARCHAR(100) NOT NULL,
    region              NVARCHAR(50) NOT NULL DEFAULT 'uaenorth',
    created_at          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    suspended_at        DATETIME2 NULL,
    terminated_at       DATETIME2 NULL
);

-- Subscription management
CREATE TABLE subscriptions (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    plan                NVARCHAR(50) NOT NULL,
    status              NVARCHAR(20) NOT NULL,        -- Active | PastDue | Cancelled
    stripe_subscription_id NVARCHAR(100) NULL,
    current_period_start DATETIME2 NOT NULL,
    current_period_end  DATETIME2 NOT NULL,
    max_seats           INT NOT NULL,
    max_storage_gb      INT NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- Subscription plans (reference)
CREATE TABLE subscription_plans (
    id                  NVARCHAR(50) PRIMARY KEY,     -- Starter, Professional, Enterprise
    display_name        NVARCHAR(100) NOT NULL,
    max_seats           INT NOT NULL,
    max_storage_gb      INT NOT NULL,
    price_monthly_usd   DECIMAL(10,2) NOT NULL,
    features            NVARCHAR(MAX) NOT NULL         -- JSON feature flags
);
```

### 5.2 Tenant Database Schema (Per-Tenant — `legalsaas-tenant-{slug}`)

Each tenant has an isolated Azure SQL Database with the following schema:

```sql
-- ═══════════════════════════════════════
-- USERS & ROLES
-- ═══════════════════════════════════════

CREATE TABLE roles (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    name            NVARCHAR(50) NOT NULL UNIQUE,       -- TenantAdmin, SeniorLawyer, Lawyer, Paralegal, ReadOnly
    display_name    NVARCHAR(100) NOT NULL,
    permissions     NVARCHAR(MAX) NOT NULL,              -- JSON array of permission keys
    is_system       BIT NOT NULL DEFAULT 0,              -- System roles can't be deleted
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE users (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    b2c_object_id   NVARCHAR(100) NOT NULL UNIQUE,       -- Azure AD B2C object ID
    email           NVARCHAR(256) NOT NULL UNIQUE,
    full_name       NVARCHAR(200) NOT NULL,
    phone           NVARCHAR(30) NULL,
    role_id         UNIQUEIDENTIFIER NOT NULL REFERENCES roles(id),
    status          NVARCHAR(20) NOT NULL DEFAULT 'Active',   -- Active | Inactive | Suspended
    is_seat_assigned BIT NOT NULL DEFAULT 1,
    last_login_at   DATETIME2 NULL,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE permission_overrides (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id         UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    permission_key  NVARCHAR(100) NOT NULL,
    granted         BIT NOT NULL,                         -- true = grant, false = deny
    UNIQUE(user_id, permission_key)
);

-- ═══════════════════════════════════════
-- CUSTOMERS
-- ═══════════════════════════════════════

CREATE TABLE customers (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    type            NVARCHAR(20) NOT NULL,               -- Individual | Company
    display_name    NVARCHAR(300) NOT NULL,
    email           NVARCHAR(256) NULL,
    phone           NVARCHAR(30) NULL,
    address         NVARCHAR(500) NULL,
    city            NVARCHAR(100) NULL,
    country         NVARCHAR(100) NULL,
    -- Company-specific
    company_name    NVARCHAR(300) NULL,
    registration_no NVARCHAR(100) NULL,
    primary_contact_name NVARCHAR(200) NULL,
    primary_contact_title NVARCHAR(100) NULL,
    -- Assignment
    assigned_user_id UNIQUEIDENTIFIER NULL REFERENCES users(id),
    status          NVARCHAR(20) NOT NULL DEFAULT 'Active',  -- Active | Prospect | Closed
    notes           NVARCHAR(MAX) NULL,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE INDEX IX_customers_assigned ON customers(assigned_user_id);
CREATE INDEX IX_customers_status ON customers(status);

-- ═══════════════════════════════════════
-- COURTS
-- ═══════════════════════════════════════

CREATE TABLE courts (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    name            NVARCHAR(200) NOT NULL,
    jurisdiction_type NVARCHAR(50) NOT NULL,             -- Commercial, Civil, Administrative, Employment, Family, Criminal, IP
    city            NVARCHAR(100) NOT NULL,
    country         NVARCHAR(100) NOT NULL,
    address         NVARCHAR(500) NULL,
    status          NVARCHAR(20) NOT NULL DEFAULT 'Active',
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ═══════════════════════════════════════
-- CASES
-- ═══════════════════════════════════════

CREATE TABLE cases (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    case_number     NVARCHAR(20) NOT NULL UNIQUE,        -- Auto: C-{YYYY}-{SEQ}
    title           NVARCHAR(500) NOT NULL,
    description     NVARCHAR(MAX) NULL,
    customer_id     UNIQUEIDENTIFIER NOT NULL REFERENCES customers(id),
    court_id        UNIQUEIDENTIFIER NULL REFERENCES courts(id),
    assigned_user_id UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    status          NVARCHAR(30) NOT NULL DEFAULT 'Intake',
        -- Intake | InProgress | Filed | AwaitingJudgment | Judgment | Closed | Archived
    priority        NVARCHAR(10) NOT NULL DEFAULT 'Normal',  -- Low | Normal | High | Urgent
    opened_at       DATE NOT NULL DEFAULT CAST(SYSUTCDATETIME() AS DATE),
    closed_at       DATE NULL,
    next_hearing_at DATETIME2 NULL,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE INDEX IX_cases_customer ON cases(customer_id);
CREATE INDEX IX_cases_assigned ON cases(assigned_user_id);
CREATE INDEX IX_cases_status ON cases(status);
CREATE INDEX IX_cases_court ON cases(court_id);

CREATE TABLE case_status_history (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    case_id         UNIQUEIDENTIFIER NOT NULL REFERENCES cases(id),
    from_status     NVARCHAR(30) NULL,
    to_status       NVARCHAR(30) NOT NULL,
    changed_by_id   UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    notes           NVARCHAR(500) NULL,
    changed_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ═══════════════════════════════════════
-- CASE CALENDAR / APPOINTMENTS
-- ═══════════════════════════════════════

CREATE TABLE appointments (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    case_id         UNIQUEIDENTIFIER NULL REFERENCES cases(id),
    title           NVARCHAR(300) NOT NULL,
    type            NVARCHAR(30) NOT NULL,               -- Hearing | Deadline | Meeting | Reminder
    starts_at       DATETIME2 NOT NULL,
    ends_at         DATETIME2 NULL,
    location        NVARCHAR(300) NULL,
    notes           NVARCHAR(MAX) NULL,
    created_by_id   UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE INDEX IX_appointments_case ON appointments(case_id);
CREATE INDEX IX_appointments_date ON appointments(starts_at);

-- ═══════════════════════════════════════
-- CASE EXPENSES
-- ═══════════════════════════════════════

CREATE TABLE case_expenses (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    case_id         UNIQUEIDENTIFIER NOT NULL REFERENCES cases(id),
    category        NVARCHAR(100) NOT NULL,              -- CourtFees, ExpertWitness, Travel, Filing, Other
    description     NVARCHAR(500) NULL,
    amount          DECIMAL(18,2) NOT NULL,
    currency        NVARCHAR(3) NOT NULL DEFAULT 'EGP',
    incurred_at     DATE NOT NULL,
    recorded_by_id  UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ═══════════════════════════════════════
-- DOCUMENTS
-- ═══════════════════════════════════════

CREATE TABLE documents (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    file_name       NVARCHAR(300) NOT NULL,
    blob_path       NVARCHAR(500) NOT NULL,              -- tenant container relative path
    content_type    NVARCHAR(100) NOT NULL,               -- MIME type
    size_bytes      BIGINT NOT NULL,
    category        NVARCHAR(50) NOT NULL,               -- Evidence, Pleadings, Contracts, Identity, PowerOfAttorney, Other
    access_level    NVARCHAR(20) NOT NULL DEFAULT 'Team', -- Private | Team | Tenant
    version         INT NOT NULL DEFAULT 1,
    description     NVARCHAR(500) NULL,
    virus_scan_status NVARCHAR(20) NOT NULL DEFAULT 'Pending', -- Pending | Clean | Infected
    -- Linkage (polymorphic)
    linked_entity_type NVARCHAR(20) NULL,                -- Case | Customer | NULL
    linked_entity_id   UNIQUEIDENTIFIER NULL,
    -- Audit
    uploaded_by_id  UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    deleted_at      DATETIME2 NULL                        -- Soft delete
);

CREATE INDEX IX_documents_linked ON documents(linked_entity_type, linked_entity_id);
CREATE INDEX IX_documents_category ON documents(category);
CREATE INDEX IX_documents_uploaded ON documents(uploaded_by_id);

CREATE TABLE document_access_log (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    document_id     UNIQUEIDENTIFIER NOT NULL REFERENCES documents(id),
    user_id         UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    action          NVARCHAR(20) NOT NULL,               -- View | Download | Delete
    ip_address      NVARCHAR(45) NULL,
    accessed_at     DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ═══════════════════════════════════════
-- AUDIT LOG
-- ═══════════════════════════════════════

CREATE TABLE audit_logs (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id         UNIQUEIDENTIFIER NULL REFERENCES users(id),
    action          NVARCHAR(20) NOT NULL,               -- Create | Update | Delete | View | Upload | Config | Login
    resource_type   NVARCHAR(50) NOT NULL,               -- User, Case, Customer, Document, Court, Settings
    resource_id     NVARCHAR(100) NULL,
    description     NVARCHAR(500) NULL,
    ip_address      NVARCHAR(45) NULL,
    user_agent      NVARCHAR(500) NULL,
    metadata        NVARCHAR(MAX) NULL,                   -- JSON with change details
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE INDEX IX_audit_user ON audit_logs(user_id);
CREATE INDEX IX_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX IX_audit_date ON audit_logs(created_at DESC);

-- ═══════════════════════════════════════
-- TENANT SETTINGS
-- ═══════════════════════════════════════

CREATE TABLE tenant_settings (
    key             NVARCHAR(100) PRIMARY KEY,
    value           NVARCHAR(MAX) NOT NULL,
    updated_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### 5.3 Subscription Plans

| Plan | Seats | Storage | Price (USD/mo) | Features |
|---|---|---|---|---|
| **Starter** | 5 | 10 GB | $49 | Cases, Customers, Documents, Calendar |
| **Professional** | 15 | 50 GB | $149 | + Audit logs, Backup/Restore, Advanced roles |
| **Enterprise** | 50 | 200 GB | $399 | + Full-text search, API access, Priority support, Custom branding |

---

## 6. Identity, Authentication & RBAC

### 6.1 Authentication Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Azure AD B2C Tenant                     │
│                                                            │
│  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │ Sign-Up / Sign-In│  │ Custom Policy: LegalSaaS     │   │
│  │ User Flow        │  │                               │   │
│  │                  │  │ Claims:                       │   │
│  │ - Email + PW     │  │   sub (object_id)            │   │
│  │ - Google SSO     │  │   email                      │   │
│  │ - Microsoft SSO  │  │   name                       │   │
│  │                  │  │   extension_tenantId          │   │
│  └─────────────────┘  │   extension_tenantSlug        │   │
│                        │   extension_role              │   │
│                        └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼  JWT (id_token / access_token)
                    ┌──────────────────┐
                    │  Tenant Portal   │
                    │  SPA (MSAL.js)   │
                    └────────┬─────────┘
                             │ Bearer Token
                             ▼
                    ┌──────────────────┐
                    │  .NET 8 API      │
                    │  JWT Validation  │
                    │  + TenantContext │
                    │  + RBAC Check    │
                    └──────────────────┘
```

**Admin Portal** uses **Microsoft Entra ID** (corporate) with Conditional Access policies:
- MFA required
- Compliant device required
- Named location restriction (office VPN or approved IPs)

### 6.2 Token Claims (JWT)

```json
{
  "sub": "a1b2c3d4-...",
  "email": "m.rashid@almansourlaw.com",
  "name": "Mohamed Rashid",
  "extension_tenantId": "7f8e9d0c-...",
  "extension_tenantSlug": "almansour",
  "extension_role": "Lawyer",
  "iss": "https://legalsaasb2c.b2clogin.com/.../v2.0/",
  "aud": "api-client-id",
  "exp": 1740000000,
  "iat": 1739996400
}
```

### 6.3 RBAC Model

**System Roles (Seeded):**

| Role | Scope | Key Permissions |
|---|---|---|
| **Tenant Admin** | Full tenant | Users, Seats, Roles, Settings, Backup, Audit, all CRUD |
| **Senior Lawyer** | Workspace | All Cases/Customers/Docs (own + team), Calendar, Assign |
| **Lawyer** | Workspace | Own Cases/Customers, Team Docs, Calendar |
| **Paralegal** | Workspace (limited) | View Cases, Upload Docs (assigned only) |
| **Read Only** | View only | View everything, no create/edit/delete |

**Permission Keys:**

```
users.view, users.create, users.edit, users.delete, users.invite
roles.view, roles.manage
seats.view, seats.manage
settings.view, settings.manage
courts.view, courts.create, courts.edit, courts.delete
backup.view, backup.request_restore
audit.view, audit.export

customers.view_all, customers.view_assigned, customers.create, customers.edit, customers.delete, customers.export
cases.view_all, cases.view_assigned, cases.create, cases.edit, cases.delete, cases.assign, cases.change_status
documents.view_team, documents.view_private, documents.upload, documents.download, documents.delete
calendar.view, calendar.create, calendar.edit, calendar.delete
expenses.view, expenses.create, expenses.edit, expenses.delete
```

**Permission Resolution Order:**
1. Check `permission_overrides` table for user-specific grant/deny
2. If no override, check `roles.permissions` JSON array for the user's role
3. Deny by default

### 6.4 Admin Portal Roles

| Role | Scope | Permissions |
|---|---|---|
| **Platform Admin** | Full platform | All operations, tenant provisioning, billing |
| **Support Agent** | Read + limited write | View tenants, impersonate, resolve tickets |
| **Billing Operator** | Financial | Subscriptions, payments, invoices |
| **Auditor** | Read-only | View all data, export reports |

---

## 7. API Layer Design

### 7.1 API Architecture

- **Style:** RESTful, resource-oriented
- **Versioning:** URL path versioning (`/api/v1/`)
- **Format:** JSON (camelCase property naming)
- **Authentication:** Bearer JWT (Azure AD B2C tokens)
- **Tenant Context:** Required header `X-Tenant-Slug` on every request (validated against JWT claim)
- **Rate Limiting:** 100 req/min per user, 1000 req/min per tenant
- **Pagination:** Cursor-based (`?cursor=xxx&limit=25`)
- **Filtering:** OData-style (`?$filter=status eq 'Active'`)
- **Sorting:** `?$orderby=created_at desc`

### 7.2 Tenant API Endpoints

```
Base URL: https://api.legalsaas.com/api/v1
Required Headers: Authorization: Bearer {token}, X-Tenant-Slug: {slug}

═══ DASHBOARD ═══
GET    /dashboard/summary                    → Dashboard stats

═══ USERS (Tenant Admin) ═══
GET    /users                                → List users (paginated)
GET    /users/{id}                           → Get user detail
POST   /users/invite                         → Invite new user
PUT    /users/{id}                           → Update user profile/role
DELETE /users/{id}                           → Deactivate user (soft)
PUT    /users/{id}/permissions               → Set permission overrides

═══ ROLES (Tenant Admin) ═══
GET    /roles                                → List roles
GET    /roles/{id}                           → Get role with permissions
POST   /roles                                → Create custom role
PUT    /roles/{id}                           → Update role permissions

═══ CUSTOMERS ═══
GET    /customers                            → List customers (paginated, filtered)
GET    /customers/{id}                       → Get customer detail
POST   /customers                            → Create customer
PUT    /customers/{id}                       → Update customer
DELETE /customers/{id}                       → Archive customer (soft delete)
GET    /customers/{id}/cases                 → List cases for customer
GET    /customers/{id}/documents             → List documents for customer

═══ CASES ═══
GET    /cases                                → List cases (paginated, filtered, sorted)
GET    /cases/{id}                           → Get case detail (summary, docs, events, expenses)
POST   /cases                                → Create case
PUT    /cases/{id}                           → Update case
PUT    /cases/{id}/status                    → Change case status
DELETE /cases/{id}                           → Archive case (soft delete)
GET    /cases/{id}/documents                 → List case documents
GET    /cases/{id}/appointments              → List case appointments
GET    /cases/{id}/expenses                  → List case expenses
GET    /cases/{id}/status-history            → Status change history

═══ DOCUMENTS ═══
GET    /documents                            → List documents (paginated, filtered)
GET    /documents/{id}                       → Get document metadata
POST   /documents/upload-url                 → Get pre-signed upload URL (SAS)
POST   /documents                            → Register uploaded document metadata
GET    /documents/{id}/download-url          → Get time-limited download URL (SAS, 15 min)
DELETE /documents/{id}                       → Soft delete document
GET    /documents/{id}/access-log            → View document access history

═══ CALENDAR / APPOINTMENTS ═══
GET    /appointments                         → List appointments (date range, filters)
GET    /appointments/{id}                    → Get appointment detail
POST   /appointments                         → Create appointment
PUT    /appointments/{id}                    → Update appointment
DELETE /appointments/{id}                    → Delete appointment

═══ EXPENSES ═══
POST   /cases/{id}/expenses                  → Add expense to case
PUT    /expenses/{id}                        → Update expense
DELETE /expenses/{id}                        → Delete expense

═══ COURTS (Tenant Admin) ═══
GET    /courts                               → List courts
POST   /courts                               → Add court
PUT    /courts/{id}                          → Update court
DELETE /courts/{id}                          → Deactivate court

═══ AUDIT LOGS (Tenant Admin) ═══
GET    /audit-logs                           → List audit logs (paginated, filtered)
GET    /audit-logs/export                    → Export as CSV

═══ SETTINGS (Tenant Admin) ═══
GET    /settings                             → Get all tenant settings
PUT    /settings                             → Update settings

═══ BACKUP (Tenant Admin) ═══
GET    /backup/status                        → Get backup status & history
POST   /backup/restore-request               → Submit restore request

═══ ONBOARDING ═══
GET    /onboarding/status                    → Get onboarding completion state
PUT    /onboarding/firm-profile              → Step 1: Set firm profile
POST   /onboarding/invite-users              → Step 2: Batch invite
PUT    /onboarding/courts                    → Step 3: Configure courts
POST   /onboarding/complete                  → Step 4: Mark complete
```

### 7.3 Admin API Endpoints

```
Base URL: https://admin-api.legalsaas.com/api/v1
Auth: Microsoft Entra ID Bearer token (internal)

═══ TENANTS ═══
GET    /tenants                              → List all tenants (paginated, filtered)
GET    /tenants/{id}                         → Tenant detail (usage, subscription, health)
POST   /tenants                              → Provision new tenant
PUT    /tenants/{id}                         → Update tenant (plan, limits, status)
PUT    /tenants/{id}/suspend                 → Suspend tenant
PUT    /tenants/{id}/reactivate              → Reactivate tenant
DELETE /tenants/{id}                         → Terminate tenant (schedule deletion)

═══ SUBSCRIPTIONS ═══
GET    /subscriptions                        → List all subscriptions
PUT    /subscriptions/{id}/approve           → Approve pending subscription
POST   /subscriptions/{id}/override          → Override plan limits

═══ MONITORING ═══
GET    /monitoring/health                    → Platform health dashboard data
GET    /monitoring/tenants/{id}/usage        → Tenant resource usage
GET    /monitoring/alerts                    → Active alerts

═══ AUDIT ═══
GET    /audit/platform                       → Platform-wide audit log
GET    /audit/tenants/{id}                   → Tenant-specific audit (support view)
```

### 7.4 API Error Response Format

```json
{
  "error": {
    "code": "CASE_NOT_FOUND",
    "message": "The requested case does not exist.",
    "target": "caseId",
    "details": [],
    "traceId": "abc123-def456"
  }
}
```

**Standard error codes:** `VALIDATION_ERROR`, `NOT_FOUND`, `FORBIDDEN`, `TENANT_SUSPENDED`, `RATE_LIMITED`, `SEAT_LIMIT_EXCEEDED`, `STORAGE_LIMIT_EXCEEDED`, `INTERNAL_ERROR`.

---

## 8. Azure Blob Storage Integration

### 8.1 Container Strategy

```
Storage Account: legalsaasdocs{env}
│
├── tenant-{tenant_id}/
│   ├── cases/
│   │   ├── {case_id}/
│   │   │   ├── {document_id}_v1_filename.pdf
│   │   │   └── {document_id}_v2_filename.pdf
│   │   └── ...
│   ├── customers/
│   │   ├── {customer_id}/
│   │   │   └── {document_id}_v1_filename.jpg
│   │   └── ...
│   └── general/
│       └── {document_id}_v1_filename.docx
│
├── tenant-{tenant_id_2}/
│   └── ... (same structure)
│
└── platform/
    ├── templates/    (document templates)
    └── exports/      (temporary export files)
```

### 8.2 Upload Flow (Direct-to-Blob with SAS)

```
1. Client → POST /api/v1/documents/upload-url
   Body: { fileName, contentType, category, linkedEntityType, linkedEntityId }

2. API validates:
   - User has documents.upload permission
   - File extension is allowed (PDF, DOC, DOCX, JPG, PNG, XLSX)
   - Tenant storage quota not exceeded
   - Returns:
     {
       "uploadUrl": "https://legalsaasdocs.blob.core.windows.net/tenant-xxx/.../file.pdf?sv=...&sig=...&se=...",
       "documentId": "new-guid",
       "expiresAt": "2026-02-18T10:15:00Z"
     }

3. Client → PUT {uploadUrl} with file body (direct to Blob, bypasses API)
   Headers: x-ms-blob-type: BlockBlob, Content-Type: application/pdf

4. Client → POST /api/v1/documents
   Body: { documentId, fileName, category, accessLevel, description, linkedEntityType, linkedEntityId, sizeBytes }

5. API:
   - Verifies blob exists at expected path
   - Creates document record in tenant DB
   - Triggers virus scan (Azure Defender for Storage or ClamAV function)
   - Logs audit event
   - Returns 201 with document metadata
```

### 8.3 Download Flow (Time-Limited SAS)

```
1. Client → GET /api/v1/documents/{id}/download-url

2. API validates:
   - User has documents.download permission
   - Document access_level matches user scope (Private → owner only, Team → all lawyers, Tenant → all)
   - Generates read-only SAS token (15-minute TTL)
   - Logs access in document_access_log
   - Returns:
     {
       "downloadUrl": "https://legalsaasdocs.blob.core.windows.net/tenant-xxx/.../file.pdf?sv=...&sig=...&se=...",
       "expiresAt": "2026-02-18T10:15:00Z",
       "fileName": "Customs_Seizure_Notice.pdf",
       "contentType": "application/pdf"
     }

3. Client opens URL in new tab or streams via fetch
```

### 8.4 Storage Security

| Control | Implementation |
|---|---|
| **Encryption at rest** | AES-256 (Azure SSE, Microsoft-managed keys; V2: CMK via Key Vault) |
| **Encryption in transit** | TLS 1.2+ enforced |
| **Access control** | No public access; SAS tokens only (generated server-side) |
| **SAS token scope** | Per-blob, read-only or write-only, 15-min expiry for downloads, 30-min for uploads |
| **Virus scanning** | Azure Defender for Storage (auto-scan on upload) |
| **Soft delete** | Enabled, 30-day retention |
| **Versioning** | Blob versioning enabled for document version history |
| **Lifecycle policy** | Move to Cool tier after 90 days inactive; Archive after 1 year |
| **Immutability** | Legal hold capability for compliance (per-blob) |
| **Network** | Private endpoint; no public blob endpoint |
| **CORS** | Allowed origins: `*.legalsaas.com` only |

---

## 9. Environment Separation & DevOps

### 9.1 Environment Strategy

| Environment | Purpose | Azure Subscription | URL Pattern | Data |
|---|---|---|---|---|
| **Local** | Developer machines | N/A | `localhost:*` | Docker SQL + Azurite emulator |
| **Dev** | Feature development | Dev subscription | `*.dev.legalsaas.com` | Synthetic test data |
| **Staging** | Pre-release validation | Dev subscription | `*.staging.legalsaas.com` | Anonymized production clone |
| **Production** | Live customers | Prod subscription | `*.legalsaas.com` | Real tenant data |

### 9.2 Infrastructure as Code (Bicep)

```
infra/
├── main.bicep                  # Orchestrator
├── modules/
│   ├── networking.bicep        # VNet, NSGs, Private Endpoints
│   ├── sql-server.bicep        # SQL Server + Elastic Pool
│   ├── sql-platform-db.bicep   # Platform database
│   ├── storage-account.bicep   # Blob storage
│   ├── app-service.bicep       # API App Service
│   ├── static-web-app.bicep    # SPA hosting
│   ├── redis.bicep             # Cache
│   ├── key-vault.bicep         # Secrets
│   ├── service-bus.bicep       # Messaging
│   ├── front-door.bicep        # CDN + WAF
│   ├── monitoring.bicep        # App Insights + Log Analytics
│   └── b2c-config.bicep        # B2C tenant config
├── parameters/
│   ├── dev.bicepparam
│   ├── staging.bicepparam
│   └── prod.bicepparam
└── tenant-provisioning/
    └── new-tenant.bicep        # Per-tenant database creation
```

### 9.3 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy-api.yml
name: Deploy Tenant API

on:
  push:
    branches: [main]
    paths: ['src/api/**']

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet restore
      - run: dotnet build --no-restore
      - run: dotnet test --no-build --verbosity normal
      - run: dotnet publish -c Release -o ./publish
      - uses: actions/upload-artifact@v4
        with: { name: api, path: ./publish }

  deploy-staging:
    needs: build-test
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
      - uses: azure/webapps-deploy@v3
        with:
          app-name: legalsaas-api-staging
          package: api

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval
    steps:
      - uses: actions/download-artifact@v4
      - uses: azure/webapps-deploy@v3
        with:
          app-name: legalsaas-api-prod
          package: api
```

**Pipeline Matrix:**

| Component | Trigger | Build | Test | Deploy Staging | Approve | Deploy Prod |
|---|---|---|---|---|---|---|
| Tenant API | Push to `main` (src/api) | .NET 8 build | xUnit + Integration | Auto | Manual gate | Auto |
| Tenant SPA | Push to `main` (src/tenant-portal) | Vite build | Vitest + Playwright | Auto | Manual gate | Auto |
| Admin API | Push to `main` (src/admin-api) | .NET 8 build | xUnit | Auto | Manual gate | Auto |
| Admin SPA | Push to `main` (src/admin-portal) | Vite build | Vitest | Auto | Manual gate | Auto |
| Public Website | Push to `main` (src/public-site) | Next.js build | — | Auto | Manual gate | Auto |
| Infrastructure | Push to `main` (infra) | Bicep what-if | — | Auto (what-if) | Manual gate | Apply |

### 9.4 Database Migrations Strategy

- **Tool:** EF Core Migrations
- **Platform DB:** Migrations in `src/api/Migrations/Platform/`
- **Tenant DB:** Migrations in `src/api/Migrations/Tenant/`
- **Deployment:** On API startup, auto-migrate Platform DB. Tenant DBs migrated via background job (rolling migration across all tenant databases).
- **Rollback:** Down migrations available; tested in staging before production.

---

## 10. Security Controls

### 10.1 Security Architecture

| Layer | Control | Implementation |
|---|---|---|
| **Edge** | DDoS Protection | Azure DDoS Protection (Standard) on VNet |
| **Edge** | WAF | Azure Front Door WAF, OWASP 3.2 ruleset |
| **Edge** | Rate Limiting | Azure Front Door + Redis-backed token bucket |
| **Transport** | TLS | TLS 1.2+ enforced everywhere; HSTS enabled |
| **Authentication** | OAuth 2.0 / OIDC | Azure AD B2C (tenants), Entra ID (admin) |
| **Authorization** | RBAC | Custom role + permission model, per-request check |
| **Tenant Isolation** | Middleware | Every API request scoped to tenant; cross-tenant access impossible |
| **Data at Rest** | Encryption | Azure SQL TDE; Blob SSE (AES-256) |
| **Data in Transit** | Encryption | TLS 1.2 between all components |
| **Secrets** | Management | Azure Key Vault; no secrets in code or config |
| **Input Validation** | Server-side | FluentValidation on all API models |
| **SQL Injection** | Prevention | EF Core parameterized queries only |
| **XSS** | Prevention | React DOM escaping + CSP headers |
| **CSRF** | Prevention | SameSite cookies + anti-forgery tokens |
| **File Upload** | Scanning | Azure Defender for Storage; allowed extensions whitelist |
| **Audit** | Logging | Every CUD operation logged with actor, timestamp, IP |
| **Network** | Segmentation | VNet + Private Endpoints; no public DB/Storage endpoints |

### 10.2 Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://legalsaasb2c.b2clogin.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https://legalsaasdocs*.blob.core.windows.net;
  connect-src 'self' https://api.legalsaas.com https://legalsaasb2c.b2clogin.com;
  frame-src https://legalsaasb2c.b2clogin.com;
```

### 10.3 Compliance Considerations

| Standard | Relevance | Measures |
|---|---|---|
| **GDPR** | EU users/data | Data processing agreements, right to erasure, export |
| **PDPL (Saudi)** | KSA users | Data residency in-region, consent management |
| **DIFC Data Protection** | UAE/DIFC firms | Compliant processing, breach notification |
| **SOC 2 Type II** | Enterprise trust (V2) | Audit logging, access controls, monitoring |
| **ISO 27001** | Information security (V2) | ISMS framework alignment |

---

## 11. Observability, Monitoring & Logging

### 11.1 Observability Stack

```
┌──────────────────────────────────────────────────────┐
│                   Azure Monitor                        │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ Application │  │ Log         │  │ Azure        │ │
│  │ Insights    │  │ Analytics   │  │ Alerts       │ │
│  │ (APM)       │  │ (KQL)       │  │ (Action      │ │
│  │             │  │             │  │  Groups)     │ │
│  └─────────────┘  └─────────────┘  └──────────────┘ │
└──────────────────────────────────────────────────────┘
```

### 11.2 What We Log

| Category | Source | Retention | Destination |
|---|---|---|---|
| **Application traces** | API (Serilog → App Insights) | 90 days | Application Insights |
| **Request/Response** | API middleware | 90 days | Application Insights |
| **Dependency calls** | EF Core, HTTP, Redis, Blob | 90 days | Application Insights |
| **Business audit logs** | Audit service | 7 years (tenant DB) | Tenant SQL Database |
| **Platform audit logs** | Admin operations | 7 years | Platform SQL Database |
| **Infrastructure metrics** | Azure resources | 90 days | Azure Monitor Metrics |
| **Security events** | WAF, auth failures | 1 year | Log Analytics |
| **Blob access logs** | Storage diagnostics | 90 days | Log Analytics |

### 11.3 Structured Logging Format

```json
{
  "timestamp": "2026-02-18T09:14:02.123Z",
  "level": "Information",
  "messageTemplate": "Case {CaseId} created by {UserId} in tenant {TenantId}",
  "properties": {
    "CaseId": "c-2026-041",
    "UserId": "a1b2c3d4",
    "TenantId": "7f8e9d0c",
    "TenantSlug": "almansour",
    "CorrelationId": "req-xyz-789",
    "SourceContext": "LegalSaaS.Api.Services.CaseService",
    "ActionName": "CreateCase",
    "RequestPath": "/api/v1/cases",
    "StatusCode": 201,
    "ElapsedMs": 142
  }
}
```

**Every log entry includes:** `TenantId`, `TenantSlug`, `UserId`, `CorrelationId` (from X-Correlation-Id header or auto-generated).

### 11.4 Alerts

| Alert | Condition | Severity | Action |
|---|---|---|---|
| **API Error Rate** | 5xx rate > 5% over 5 min | Critical | PagerDuty + Slack |
| **API Latency** | P95 > 2s over 5 min | Warning | Slack |
| **Tenant DB High DTU** | > 80% DTU for 10 min | Warning | Slack + auto-scale eval |
| **Storage Quota Warning** | Tenant > 90% of plan storage | Info | Email tenant admin |
| **Auth Failures** | > 50 failures in 5 min per tenant | Warning | Slack + review |
| **Virus Detected** | Any document flagged infected | Critical | Auto-quarantine + alert |
| **Elastic Pool Saturation** | > 90% eDTU | Critical | PagerDuty |
| **Deployment Failure** | CI/CD pipeline failure | Warning | Slack + GitHub Issues |

### 11.5 Health Checks

```
GET /healthz          → 200 OK (basic liveness)
GET /healthz/ready    → 200 OK (checks: SQL, Redis, Blob, B2C)

Response:
{
  "status": "Healthy",
  "checks": {
    "sql-platform": "Healthy",
    "sql-tenant-pool": "Healthy",
    "redis": "Healthy",
    "blob-storage": "Healthy",
    "b2c": "Healthy"
  },
  "version": "1.2.0",
  "uptime": "5d 3h 22m"
}
```

---

## 12. UX & Design Architecture

### 12.1 Personas

| Persona | Role | Goals | Pain Points |
|---|---|---|---|
| **Ahmed (Tenant Admin)** | Managing Partner, 45 | Manage firm subscription, users, seats; oversee operations; ensure compliance | Juggling admin + legal work; needs fast setup |
| **Mohamed (Lawyer)** | Associate Lawyer, 32 | Manage assigned cases, track hearings, organize documents | Paper-based chaos; missed deadlines |
| **Layla (Senior Lawyer)** | Senior Associate, 38 | Supervise team cases, review documents, manage clients | No visibility across team workload |
| **Karim (Paralegal)** | Paralegal, 26 | Upload documents, prepare case files, schedule appointments | Limited system access; manual file management |
| **Fatima (Admin Operator)** | SaaS Platform Admin, 30 | Provision tenants, monitor health, manage billing | Manual provisioning; no unified dashboard |

### 12.2 Design System — Finalized Tokens

| Token | Value | Usage |
|---|---|---|
| **Font — Display** | Libre Baskerville | Headings, entity names, stats — conveys legal authority |
| **Font — Body** | IBM Plex Sans | All body text, labels, navigation — clarity and readability |
| **Font — Mono** | IBM Plex Mono | IDs, timestamps, technical data |
| **Ink (Primary)** | `#0F1923` | Primary text |
| **Ink (Secondary)** | `#2C3A4A` | Secondary text |
| **Ink (Tertiary)** | `#536172` | Tertiary text |
| **Muted** | `#8C9BAB` | Labels, placeholders |
| **Border** | `#D8E0E8` | Dividers, card borders |
| **Surface** | `#F4F7FA` | Page backgrounds |
| **White** | `#FFFFFF` | Card backgrounds |
| **Blue (Primary)** | `#1A56DB` | Action, links, selected states |
| **Teal** | `#0E7490` | Accent, secondary roles |
| **Green** | `#057A55` | Success, active status |
| **Amber** | `#B45309` | Warning, pending |
| **Red** | `#B91C1C` | Error, danger, urgent |
| **Purple** | `#5B21B6` | Accent, document countR |
| **Chrome** | `#141E2B` | App chrome, sidebar backgrounds |
| **Border Radius** | 5px (buttons, inputs) / 6px (cards) / 8px (entity icons) | |
| **Spacing Scale** | 4, 8, 10, 12, 14, 16, 20, 24, 28, 32, 40 | |

### 12.3 Information Architecture — Tenant Portal

```
Tenant Portal (almansour.legalsaas.com)
│
├── Top Navigation (horizontal)
│   ├── Dashboard
│   ├── Customers
│   ├── Cases
│   ├── Documents
│   └── Calendar
│
├── Left Sidebar (contextual per section)
│   ├── Section-specific navigation
│   ├── Quick filters
│   └── Sub-navigation
│
└── Workspace Area (content)
    ├── Dashboard → Stats + Activity + Seat Overview
    ├── Customers → Master List → Detail (Profile / Docs / Cases / Payments)
    ├── Cases → Kanban Board | List View → Detail (Overview / Docs / Calendar / Expenses / History)
    ├── Documents → Tree + Master List → Detail (Preview / Metadata / Access Log)
    ├── Calendar → Month / Week / Agenda views
    │
    └── Admin Section (visible to Tenant Admin only)
        ├── Users → Master List → Detail (Profile / Permissions)
        ├── Seats → Allocation overview
        ├── Roles → Role editor
        ├── Courts → Master List → Detail (Court form)
        ├── Backup → Status + Restore request
        ├── Audit Logs → Filterable table
        └── Onboarding Wizard (first login)
```

### 12.4 Navigation Model

**Dual-axis Navigation:**
- **Horizontal Top Bar:** Primary module switching (Dashboard, Customers, Cases, Documents, Calendar)
- **Vertical Left Sidebar (224px):** Contextual navigation within each module + quick filters

**Pattern Rationale:** Law firm users work deeply within one module (e.g., Cases) for extended periods, then switch modules. The top bar enables fast module switching while the sidebar provides depth within the current context.

### 12.5 Enterprise UX Patterns

| Pattern | Usage | Screen References |
|---|---|---|
| **Master-Detail (3-panel)** | Primary data exploration: Users, Customers, Cases, Documents, Courts | TA-02, CM-01, CASE-02, DOC-01, TA-05 |
| **Dashboard** | KPI overview with stat cards, charts, activity feed | TA-01 |
| **Kanban Board** | Visual case pipeline management | CASE-01 |
| **Calendar** | Month/Week/Agenda with event color coding | CASE-05 |
| **Step Wizard** | Guided onboarding flow (4 steps) | TA-08 |
| **Data Table** | Audit logs, tabular data with filters + export | TA-07 |
| **Upload Flow** | Drag-drop → metadata form → confirm | DOC-03 |
| **Entity Detail Card** | Hero section with icon, name, badges, metadata | All detail views |
| **Tabbed Content** | Sub-navigation within detail views (Overview, Documents, etc.) | CM-02, CASE-02 |
| **Stat Cards** | KPI display with value, label, delta indicator | TA-01 |
| **Activity Feed** | Chronological action log with avatars | TA-01 |
| **Breadcrumb** | Location context in detail views | All detail views |
| **Badge System** | Status, role, category, access level indicators | Throughout |

### 12.6 Wireframe Inventory — Tenant Portal (Figma-Ready)

The following wireframes are fully specified in [LegalSaaS_Wireframes_v2.html](doc/LegalSaaS_Wireframes_v2.html) with pixel-accurate layouts, design tokens, and interactive navigation.

| Screen ID | Screen Name | Layout Pattern | Status |
|---|---|---|---|
| **TA-01** | Tenant Admin — Dashboard | Dashboard (Stats + Activity + Seat Donut) | Complete |
| **TA-02** | Tenant Admin — User Management | Master-Detail (user list → user form) | Complete |
| **TA-05** | Tenant Admin — Courts Configuration | Master-Detail (court list → court form) | Complete |
| **TA-06** | Tenant Admin — Backup & Restore | Full-width (status cards + table + form) | Complete |
| **TA-07** | Tenant Admin — Audit Logs | Full-width filterable data table | Complete |
| **TA-08** | Tenant Admin — Onboarding Wizard | Centered wizard (4-step) | Complete |
| **CM-01** | Customer — Customer List | Master-Detail (customer list → profile) | Complete |
| **CM-02** | Customer — Profile + Documents | Detail with tabs (Overview, Docs, Cases, Payments) | Complete |
| **CASE-01** | Cases — Kanban Board | Kanban (5 columns: Intake → Closed) | Complete |
| **CASE-02** | Cases — Case Detail | Master-Detail (case list → detail with tabs) | Complete |
| **CASE-05** | Cases — Calendar | Calendar (Month view + sidebar) | Complete |
| **DOC-01** | Documents — Library | Tree + Master-Detail (3-panel) | Complete |
| **DOC-03** | Documents — Upload | Centered form (drag-drop + metadata) | Complete |

**Figma Translation Notes:**
- All screens use 1440px width canvas
- All design token values are specified in Section 12.2
- Component primitives (buttons, badges, inputs, tables, cards) are defined in the CSS design system
- Spacing, typography, and color usage is consistent across all wireframes
- Interactive states (hover, active, focus, selected) are defined in CSS

### 12.7 Responsive Strategy

| Breakpoint | Width | Behavior |
|---|---|---|
| **Desktop** | ≥ 1440px | Full 3-panel master-detail |
| **Desktop (compact)** | 1024–1439px | Sidebar collapses to icons; master panel narrows |
| **Tablet** | 768–1023px | Sidebar hidden (hamburger); master/detail stacked |
| **Mobile** | < 768px | Full-width single panel; bottom navigation | 

MVP (Wave 1) targets desktop (≥ 1024px) only. Responsive design is V2 scope.

---

## 13. Public Website (V2)

### 13.1 Purpose & Scope

The Public Website is the marketing and acquisition front for LegalSaaS. It is public-facing, SEO-optimized, and drives subscription sign-ups.

**URL:** `https://legalsaas.com`  
**Technology:** Next.js 14 (App Router) with Static Site Generation (SSG) for marketing pages + Server-Side Rendering (SSR) for dynamic content.  
**Hosting:** Azure Static Web Apps (Standard tier) with Azure CDN.

### 13.2 Page Structure

| Page | Route | Purpose |
|---|---|---|
| **Home** | `/` | Hero, value proposition, feature highlights, testimonials, CTA |
| **Features** | `/features` | Detailed feature showcase (Cases, Documents, Calendar, etc.) |
| **Pricing** | `/pricing` | Plan comparison table (Starter / Professional / Enterprise) |
| **About** | `/about` | Company story, team, mission |
| **Contact** | `/contact` | Contact form, office locations |
| **Blog** | `/blog` | Legal tech thought leadership (V2+) |
| **Sign Up** | `/signup` | Multi-step subscription flow |
| **Login** | `/login` | Redirect to tenant subdomain + B2C auth |
| **Legal** | `/terms`, `/privacy`, `/dpa` | Legal pages |

### 13.3 Subscription Flow (High-Level)

```
1. Visitor → /pricing → Selects plan → /signup
2. Sign-Up Form:
   - Step 1: Firm Information (name, country, size)
   - Step 2: Admin Account (name, email, password → Azure AD B2C registration)
   - Step 3: Choose Slug (subdomain: xxx.legalsaas.com)
   - Step 4: Payment (Stripe Checkout embedded)
   - Step 5: Confirmation → "Your workspace is being prepared..."
3. Backend:
   - Stripe webhook → POST /admin-api/v1/subscriptions/webhook
   - Admin API creates tenant record → triggers provisioning pipeline
   - Service Bus message → Tenant Provisioning Function
   - Provisioning: DB creation, migrations, seed data, blob container
   - Email sent to admin: "Your workspace is ready!"
4. Admin → clicks link → almansour.legalsaas.com → Onboarding Wizard (TA-08)
```

### 13.4 Integration with Admin Portal

- Subscription creation triggers Admin API webhook
- Pending subscriptions appear in Admin Portal for review (manual approval for Enterprise plan)
- Starter and Professional plans are auto-approved
- Admin Portal operators can override, suspend, or upgrade subscriptions

### 13.5 Public Website UX (High-Level)

- Clean, professional design aligned with LegalSaaS brand
- Dark header with `#0F1923`, serif headings (Libre Baskerville), blue CTA buttons
- Trust indicators: security badges, compliance logos, customer count
- No detailed wireframes required for V2; design follows brand guidelines

---

## 14. Tenant Portal (MVP — Wave 1)

### 14.1 Technical Architecture Summary

| Aspect | Decision |
|---|---|
| **SPA Framework** | React 18 + TypeScript + Vite |
| **Routing** | React Router v6 |
| **State** | TanStack Query (server state) + Zustand (UI state) |
| **UI Primitives** | Radix UI (accessible) + Tailwind CSS (design tokens) |
| **Icons** | Lucide React |
| **Forms** | React Hook Form + Zod validation |
| **Tables** | TanStack Table |
| **Calendar** | FullCalendar React |
| **Auth** | MSAL.js v2 (Azure AD B2C) |
| **API Client** | Axios with interceptors (auth, tenant header, error handling) |
| **Charts** | Recharts (dashboard stats) |
| **Drag & Drop** | @dnd-kit (Kanban board) |
| **File Upload** | Direct-to-blob via SAS URL |
| **Build** | Vite → optimized static assets |
| **Deployment** | Azure Static Web Apps |

### 14.2 SPA Project Structure

```
src/
├── app/
│   ├── App.tsx                    # Root component
│   ├── router.tsx                 # Route definitions
│   └── providers.tsx              # All context providers
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── services/
│   ├── dashboard/
│   │   ├── components/
│   │   │   ├── DashboardPage.tsx
│   │   │   ├── StatCard.tsx
│   │   │   ├── SeatDonut.tsx
│   │   │   └── ActivityFeed.tsx
│   │   └── hooks/
│   │       └── useDashboard.ts
│   ├── customers/
│   │   ├── components/
│   │   │   ├── CustomerListPage.tsx
│   │   │   ├── CustomerMasterList.tsx
│   │   │   ├── CustomerDetail.tsx
│   │   │   ├── CustomerForm.tsx
│   │   │   └── CustomerCases.tsx
│   │   ├── hooks/
│   │   │   ├── useCustomers.ts
│   │   │   └── useCustomer.ts
│   │   └── types.ts
│   ├── cases/
│   │   ├── components/
│   │   │   ├── CaseKanbanPage.tsx
│   │   │   ├── CaseDetailPage.tsx
│   │   │   ├── CaseForm.tsx
│   │   │   ├── KanbanBoard.tsx
│   │   │   ├── KanbanColumn.tsx
│   │   │   ├── KanbanCard.tsx
│   │   │   ├── CaseExpenses.tsx
│   │   │   └── CaseStatusHistory.tsx
│   │   ├── hooks/
│   │   └── types.ts
│   ├── documents/
│   │   ├── components/
│   │   │   ├── DocumentLibraryPage.tsx
│   │   │   ├── DocumentTree.tsx
│   │   │   ├── DocumentList.tsx
│   │   │   ├── DocumentDetail.tsx
│   │   │   ├── DocumentUploadPage.tsx
│   │   │   └── DocumentPreview.tsx
│   │   ├── hooks/
│   │   └── types.ts
│   ├── calendar/
│   │   ├── components/
│   │   │   ├── CalendarPage.tsx
│   │   │   └── AppointmentForm.tsx
│   │   └── hooks/
│   ├── admin/
│   │   ├── users/
│   │   ├── courts/
│   │   ├── backup/
│   │   ├── audit/
│   │   ├── onboarding/
│   │   └── settings/
│   └── shared/
│       ├── components/
│       │   ├── layout/
│       │   │   ├── AppShell.tsx
│       │   │   ├── TopBar.tsx
│       │   │   ├── Sidebar.tsx
│       │   │   └── MasterDetailLayout.tsx
│       │   ├── ui/
│       │   │   ├── Button.tsx
│       │   │   ├── Badge.tsx
│       │   │   ├── Card.tsx
│       │   │   ├── DataTable.tsx
│       │   │   ├── SearchInput.tsx
│       │   │   ├── EntityHeader.tsx
│       │   │   ├── Breadcrumb.tsx
│       │   │   ├── Tabs.tsx
│       │   │   ├── StatCard.tsx
│       │   │   └── UploadZone.tsx
│       │   └── feedback/
│       │       ├── ErrorBoundary.tsx
│       │       ├── LoadingSkeleton.tsx
│       │       └── EmptyState.tsx
│       ├── hooks/
│       │   ├── useTenantContext.ts
│       │   ├── usePermission.ts
│       │   └── useDebounce.ts
│       └── services/
│           ├── apiClient.ts
│           └── blobUpload.ts
├── design-tokens/
│   └── tokens.css                 # CSS custom properties (from Section 12.2)
└── types/
    └── api.ts                     # Generated API types
```

### 14.3 Tenant Isolation in Frontend

```typescript
// useTenantContext.ts
const useTenantContext = () => {
  const slug = window.location.hostname.split('.')[0]; // "almansour"
  const { data: tenant } = useQuery({
    queryKey: ['tenant', slug],
    queryFn: () => apiClient.get(`/api/v1/tenants/current`, {
      headers: { 'X-Tenant-Slug': slug }
    }),
    staleTime: 5 * 60 * 1000, // Cache 5 min
  });
  return { slug, tenant };
};

// apiClient.ts — Interceptor auto-injects tenant header
apiClient.interceptors.request.use((config) => {
  const slug = window.location.hostname.split('.')[0];
  config.headers['X-Tenant-Slug'] = slug;
  // MSAL token added by auth interceptor
  return config;
});
```

### 14.4 RBAC in Frontend

```typescript
// usePermission.ts
const usePermission = (permissionKey: string): boolean => {
  const { user } = useAuth();
  if (!user) return false;

  // Check overrides first
  const override = user.permissionOverrides?.find(o => o.key === permissionKey);
  if (override !== undefined) return override.granted;

  // Check role permissions
  return user.role.permissions.includes(permissionKey);
};

// Usage in components
const CanDeleteDoc = () => {
  const canDelete = usePermission('documents.delete');
  if (!canDelete) return null;
  return <Button variant="danger">Delete</Button>;
};
```

### 14.5 MVP Feature Matrix

| Feature | MVP (Wave 1) | V2 |
|---|---|---|
| **Dashboard** | Stats, Activity Feed, Seat Overview | Charts, Trends, Custom Widgets |
| **User Management** | CRUD, Invite, Role Assignment, Permission Overrides | Bulk import, SSO config |
| **Seat Management** | View allocation, basic management | Self-service upgrade |
| **Role Management** | System roles (5 built-in) | Custom role builder |
| **Court Configuration** | CRUD courts | Court templates per jurisdiction |
| **Onboarding Wizard** | 4-step setup | Guided tour, video tutorials |
| **Customer Management** | CRUD, Individual/Company, Master-Detail | Import from CSV, CRM integration |
| **Case Management** | Kanban + List + Detail, status flow, assignment | Advanced workflow, templates |
| **Calendar** | Month view, appointment CRUD | Week/Agenda views, iCal sync |
| **Document Management** | Upload, download (SAS), categories, access levels | Full-text search, OCR, versioning UI |
| **Document Upload** | Drag-drop, metadata, category, linking | Batch upload, template generation |
| **Expenses** | Manual expense entry per case | Receipt OCR, reports, export |
| **Audit Logs** | View + filter + CSV export | Advanced analytics, compliance reports |
| **Backup** | View status, request restore | Self-service point-in-time restore |
| **Notifications** | Badge count indicator | Real-time push, email digests |
| **Search** | In-list search (client-side filter) | Global full-text search (AI Search) |
| **Bilingual** | English only | Arabic + English (RTL support) |
| **Responsive** | Desktop only (≥ 1024px) | Tablet + Mobile |

### 14.6 Screen Specifications

#### TA-01: Dashboard

**Layout:** Full-width workspace (no master-detail)  
**Sections:**
1. **Greeting bar:** "Good morning, {name}" + firm name + plan + date + Invite User CTA
2. **Stat cards (4-col grid):** Seats Used (x/y), Active Cases, Clients, Documents (with storage)
3. **Two-column grid:**
   - Left: Seat Allocation (donut chart + user table)
   - Right: Recent Activity (avatar + action + timestamp feed)

**Data Requirements:** `GET /dashboard/summary`

#### TA-02: User Management

**Layout:** Master-Detail (380px master + flexible detail)  
**Master Panel:** User list with avatar, name, role badge, status badge, last seen  
**Detail Panel:** User form (name, email, role dropdown, status dropdown) + permission overrides section  
**Actions:** Invite New User, Save Changes, Revoke Access

#### CM-01 / CM-02: Customer Management

**Layout:** Master-Detail (380px master + flexible detail)  
**Master Panel:** Customer list with name, email, type icon, case count badge, status, assigned lawyer  
**Detail Panel:**
- Entity header (icon, name, metadata, badges)
- Tabs: Overview | Documents | Cases | Payments
- Overview tab: Contact details card + Active cases card

#### CASE-01: Case Kanban

**Layout:** Full-width with toolbar  
**Toolbar:** Title, List/Board toggle, search, court filter, New Case button  
**Board:** 5 columns (Intake → In Progress → Filed/Awaiting → Judgment → Closed)  
**Cards:** Title, client, court, assigned lawyer avatar, status badge, deadline warning

#### CASE-02: Case Detail

**Layout:** Master-Detail (300px master + flexible detail)  
**Master Panel:** Narrow case list (title, number, status badge)  
**Detail Panel:**
- Entity header (case number badge, title, status badge, metadata: court, client, lawyer, dates)
- Tabs: Overview | Documents | Calendar | Expenses | Status History
- Overview tab: Case summary + Documents list + Upcoming events + Expenses

#### CASE-05: Calendar

**Layout:** Sidebar (upcoming events) + Full calendar workspace  
**Calendar:** Monthly grid with color-coded events (blue=meeting, amber=hearing, red=deadline)  
**Toolbar:** Previous/Next month, month title, Add Appointment button

#### DOC-01: Document Library

**Layout:** Tree sidebar (240px) + Master list (400px) + Detail pane  
**Tree:** Folder hierarchy (All Documents → Cases → {case} / Customers → {customer} / Templates)  
**Master:** File list with icon, name, category badge, access badge, case/customer link  
**Detail:** Preview area (PDF/image preview placeholder) + Metadata side panel (document info, access control, security, recent access)

#### DOC-03: Document Upload

**Layout:** Centered form (max 640px)  
**Flow:** Drag-drop zone → File queue → Metadata form (category, access level, linked entity, description) → Security note → Upload button

---

## 15. Admin Portal (V2)

### 15.1 Purpose & Scope

The Admin Portal is an internal-only application for SaaS platform operators. It provides tenant lifecycle management, subscription administration, platform monitoring, and internal tooling.

**URL:** `https://admin.legalsaas.com`  
**Auth:** Microsoft Entra ID (MFA + Conditional Access required)  
**Access:** Platform Admin, Support Agent, Billing Operator, Auditor roles

### 15.2 Architecture Placement

```
                        Admin Portal SPA
                              │
                    admin.legalsaas.com
                              │
                    admin-api.legalsaas.com
                    (Azure App Service — separate from Tenant API)
                              │
               ┌──────────────┼──────────────┐
               │              │              │
        Platform DB     Tenant DBs      Blob Storage
        (read/write)    (read-only       (management
                         for support)     operations)
```

The Admin API is deployed as a **separate Azure App Service** in the same VNet but with distinct authentication (Entra ID), separate scaling, and its own deployment pipeline. It accesses the Platform DB for tenant/subscription management and can read tenant databases for support operations.

### 15.3 Integration Boundaries

| Integration | Direction | Mechanism |
|---|---|---|
| **Stripe → Admin API** | Inbound webhook | Subscription created/updated/cancelled events |
| **Admin API → Tenant DB** | Outbound | Connection string lookup → tenant database provisioning |
| **Admin API → Blob Storage** | Outbound | Container creation, storage metrics |
| **Admin API → Azure SQL** | Outbound | Database creation in Elastic Pool |
| **Admin API → Service Bus** | Outbound | Tenant provisioning events |
| **Admin API → B2C** | Outbound | Graph API for user management |
| **Public Website → Admin API** | Inbound | Subscription creation trigger |

### 15.4 Feature Outline

#### Tenant Management
- **Tenant List:** Filterable/searchable table (slug, name, plan, status, seats, created_at)
- **Tenant Detail:** Overview (subscription, usage, health), Users tab, Audit tab
- **Tenant Provisioning:** Manual creation form (bypasses Stripe for internal/trial tenants)
- **Tenant Lifecycle:** Suspend, Reactivate, Terminate (with 90-day data retention)

#### Subscription Management
- **Pending Approvals:** Queue of subscriptions awaiting manual approval (Enterprise plan)
- **Subscription Override:** Change plan, adjust limits, extend trial
- **Payment History:** Stripe webhook logs, payment status

#### Platform Monitoring
- **Health Dashboard:** Service health indicators, API latency, error rates
- **Tenant Usage:** Per-tenant metrics (storage, seats, API calls, active users)
- **Alerts Dashboard:** Active alerts, acknowledgment, resolution tracking

#### Security & Compliance
- **Platform Audit Log:** All admin operations logged
- **Tenant Impersonation:** Support agents can view (not modify) tenant data for troubleshooting
- **IP Allowlisting:** Admin Portal access restricted to approved networks

### 15.5 Admin Portal Roles (Detailed)

| Role | Permissions |
|---|---|
| **Platform Admin** | Full CRUD on tenants, subscriptions, billing; provisioning; monitoring; audit |
| **Support Agent** | Read tenants; view tenant data (impersonate read-only); create support tickets |
| **Billing Operator** | Manage subscriptions, approvals, payment overrides; no tenant data access |
| **Auditor** | Read-only access to all admin data; export capabilities |

---

## 16. Delivery & Planning

### 16.1 MVP vs V2 Scope Summary

| Capability | MVP (Wave 1) | V2 |
|---|---|---|
| Public Website | Landing page placeholder | Full marketing site + subscription flow |
| Tenant Portal | Full (desktop) | + Mobile, Bilingual, Search, Advanced workflows |
| Admin Portal | Manual provisioning scripts | Full Admin Portal UI + automated provisioning |
| Billing Integration | Manual (no Stripe) | Stripe integration + self-service |
| Search | In-list filtering | Azure AI Search (full-text) |
| Notifications | In-app badge | Push, email, SMS (V2+) |
| Branding | LegalSaaS default | Custom tenant branding (Enterprise) |
| Analytics | Basic dashboard stats | Reporting engine, exports |

### 16.2 Epics & User Stories

#### EPIC 1: Platform Foundation
| ID | Story | Priority | Points |
|---|---|---|---|
| PF-01 | As a developer, I can provision the Azure infrastructure via Bicep so the platform has a reproducible environment | Must | 8 |
| PF-02 | As a developer, I can run the API locally with Docker SQL + Azurite so I can develop without Azure access | Must | 5 |
| PF-03 | As a developer, I can deploy the API via GitHub Actions to staging and production | Must | 5 |
| PF-04 | As a developer, I can deploy the SPA via GitHub Actions to Azure Static Web Apps | Must | 3 |
| PF-05 | As a developer, the tenant resolution middleware correctly resolves tenant from subdomain and headers | Must | 5 |
| PF-06 | As an operator, health check endpoints report readiness of all dependencies | Must | 3 |
| PF-07 | As a developer, structured logging with tenant context flows to Application Insights | Must | 3 |

#### EPIC 2: Identity & RBAC
| ID | Story | Priority | Points |
|---|---|---|---|
| ID-01 | As a user, I can sign in via Azure AD B2C with email/password | Must | 8 |
| ID-02 | As a user, my JWT contains tenantId and role claims | Must | 5 |
| ID-03 | As the API, I validate JWT tokens and enforce RBAC on every endpoint | Must | 8 |
| ID-04 | As a tenant admin, I can assign roles to users | Must | 3 |
| ID-05 | As a tenant admin, I can set permission overrides per user | Should | 5 |
| ID-06 | As the SPA, I conditionally show/hide UI elements based on permissions | Must | 5 |

#### EPIC 3: Tenant Administration
| ID | Story | Priority | Points |
|---|---|---|---|
| TA-01 | As a tenant admin, I see a dashboard with seat usage, active cases, clients, and documents stats | Must | 5 |
| TA-02 | As a tenant admin, I can view and manage users in a master-detail layout | Must | 8 |
| TA-03 | As a tenant admin, I can invite new users via email | Must | 5 |
| TA-04 | As a tenant admin, I can revoke user access | Must | 3 |
| TA-05 | As a tenant admin, I can configure courts (CRUD) | Must | 5 |
| TA-06 | As a tenant admin, I can view backup status and request a restore | Should | 5 |
| TA-07 | As a tenant admin, I can view and filter audit logs | Must | 5 |
| TA-08 | As a new tenant admin, I am guided through a 4-step onboarding wizard | Must | 8 |

#### EPIC 4: Customer Management
| ID | Story | Priority | Points |
|---|---|---|---|
| CM-01 | As a lawyer, I can view a master-detail list of all customers | Must | 5 |
| CM-02 | As a lawyer, I can create a new customer (Individual or Company) | Must | 5 |
| CM-03 | As a lawyer, I can view a customer's profile with overview, documents, and cases tabs | Must | 5 |
| CM-04 | As a lawyer, I can edit customer details | Must | 3 |
| CM-05 | As a lawyer, I can filter customers by status, type, and assigned lawyer | Should | 3 |
| CM-06 | As a lawyer, I can search customers by name, email, or company | Must | 3 |

#### EPIC 5: Case Management
| ID | Story | Priority | Points |
|---|---|---|---|
| CS-01 | As a lawyer, I can view cases in a Kanban board grouped by status | Must | 8 |
| CS-02 | As a lawyer, I can drag cases between Kanban columns to change status | Should | 5 |
| CS-03 | As a lawyer, I can view a case's detail page with overview, documents, calendar, expenses, and history tabs | Must | 8 |
| CS-04 | As a lawyer, I can create a new case linked to a customer and court | Must | 5 |
| CS-05 | As a lawyer, I can update case status with notes | Must | 3 |
| CS-06 | As a lawyer, I can add expenses to a case | Must | 3 |
| CS-07 | As a lawyer, I can view case status change history | Should | 3 |
| CS-08 | As a lawyer, I can switch between Kanban and List views | Should | 3 |
| CS-09 | As a lawyer, I can filter cases by court, status, and assigned lawyer | Should | 3 |

#### EPIC 6: Document Management
| ID | Story | Priority | Points |
|---|---|---|---|
| DC-01 | As a lawyer, I can browse documents in a tree + master-detail layout | Must | 8 |
| DC-02 | As a lawyer, I can upload a document via drag-drop with metadata (category, access level, linked entity) | Must | 8 |
| DC-03 | As a lawyer, I can download a document via time-limited SAS URL | Must | 5 |
| DC-04 | As a lawyer, I can view document metadata and access log in a side panel | Must | 5 |
| DC-05 | As a lawyer, I can filter documents by type and category | Should | 3 |
| DC-06 | As a tenant admin, uploaded documents are automatically virus-scanned | Must | 5 |
| DC-07 | As the system, documents are stored in tenant-isolated blob containers | Must | 3 |

#### EPIC 7: Calendar
| ID | Story | Priority | Points |
|---|---|---|---|
| CL-01 | As a lawyer, I can view a monthly calendar with case appointments | Must | 5 |
| CL-02 | As a lawyer, I can create, edit, and delete appointments | Must | 5 |
| CL-03 | As a lawyer, I can see color-coded events (hearing, deadline, meeting) | Must | 3 |
| CL-04 | As a lawyer, I can view upcoming appointments in the sidebar | Should | 2 |

#### EPIC 8: Shared UX Components
| ID | Story | Priority | Points |
|---|---|---|---|
| UX-01 | As a user, the app shell (top bar + sidebar + workspace) renders consistently across all modules | Must | 8 |
| UX-02 | As a user, master-detail layout is reusable across Customers, Cases, Documents, Users, Courts | Must | 5 |
| UX-03 | As a user, I see loading skeletons during data fetches | Should | 3 |
| UX-04 | As a user, I see empty states when no data exists | Must | 2 |
| UX-05 | As a user, errors are displayed gracefully with retry options | Must | 3 |
| UX-06 | As a user, the design system (tokens, buttons, badges, tables) matches the approved wireframes | Must | 5 |

**Total Story Points (MVP):** ~252 points

### 16.3 Phased Roadmap

```
═══════════════════════════════════════════════════════════════════════
                    LegalSaaS — Delivery Roadmap
═══════════════════════════════════════════════════════════════════════

PHASE 0: Foundation (Weeks 1–3)
├── Azure infrastructure (Bicep)
├── CI/CD pipelines
├── API skeleton + tenant resolution middleware
├── Auth (Azure AD B2C integration)
├── SPA scaffold + design system implementation
├── Database schema + EF Core migrations
└── Local development environment

PHASE 1: Core Tenant Admin (Weeks 4–7)
├── Dashboard (TA-01)
├── User Management (TA-02)
├── Onboarding Wizard (TA-08)
├── Court Configuration (TA-05)
├── RBAC enforcement (API + frontend)
├── Audit Logging (TA-07)
└── Backup Status (TA-06)

PHASE 2: Customer & Case Management (Weeks 8–12)
├── Customer CRUD + Master-Detail (CM-01, CM-02)
├── Case CRUD + Kanban Board (CASE-01)
├── Case Detail + Tabs (CASE-02)
├── Case Status Flow
├── Expenses
└── Status History

PHASE 3: Documents & Calendar (Weeks 13–16)
├── Document Library (DOC-01)
├── Document Upload with SAS (DOC-03)
├── Document Download with SAS
├── Virus Scanning Integration
├── Calendar (CASE-05)
├── Appointment CRUD
└── Integration Testing

PHASE 4: Polish & Launch Prep (Weeks 17–19)
├── End-to-end testing
├── Performance optimization
├── Security audit
├── Penetration testing
├── User acceptance testing (3 pilot firms)
├── Documentation
└── Production deployment

MVP LAUNCH: Week 20
═══════════════════════════════════════════════════════════════════════

V2 PHASES (Post-Launch):

PHASE 5: Public Website (Weeks 21–26)
├── Marketing pages (Next.js)
├── Subscription flow
├── Stripe integration
├── Admin Portal — Tenant Provisioning
└── Automated onboarding pipeline

PHASE 6: Admin Portal (Weeks 27–34)
├── Admin SPA scaffold
├── Tenant Management screens
├── Subscription Management
├── Platform Monitoring Dashboard
├── Support Agent tools
└── Billing Operations

PHASE 7: Platform Maturity (Weeks 35–44)
├── Arabic / RTL support
├── Responsive design (tablet + mobile)
├── Azure AI Search integration
├── Notification system (push + email)
├── Advanced reporting
├── Custom tenant branding (Enterprise)
└── SOC 2 Type II preparation
```

### 16.4 Timeline & Milestones

| Milestone | Target Date | Deliverable |
|---|---|---|
| **M0: Kickoff** | Week 1 (Mar 2, 2026) | Team aligned, environments provisioned |
| **M1: Foundation Complete** | Week 3 (Mar 20, 2026) | Infrastructure + Auth + SPA scaffold working E2E |
| **M2: Admin Features** | Week 7 (Apr 17, 2026) | Dashboard, Users, Onboarding, Courts, Audit |
| **M3: Core Workflows** | Week 12 (May 22, 2026) | Customers + Cases fully functional |
| **M4: Full MVP** | Week 16 (Jun 19, 2026) | + Documents + Calendar + Integration tested |
| **M5: MVP Launch** | Week 20 (Jul 17, 2026) | Production deployment with 3 pilot tenants |
| **M6: Public Website** | Week 26 (Aug 28, 2026) | Self-service subscription live |
| **M7: Admin Portal** | Week 34 (Oct 23, 2026) | Full Admin Portal operational |
| **M8: V2 Complete** | Week 44 (Jan 1, 2027) | Bilingual, Mobile, Search, Notifications |

### 16.5 Team Composition (Recommended)

| Role | Count | Focus |
|---|---|---|
| **Tech Lead / Architect** | 1 | Architecture, code reviews, infrastructure |
| **Backend Engineer** | 1–2 | .NET 8 API, EF Core, Azure integration |
| **Frontend Engineer** | 1–2 | React SPA, design system, wireframe implementation |
| **QA Engineer** | 1 | Test strategy, automation, UAT coordination |
| **Product Owner** | 1 (part-time) | Prioritization, stakeholder management |

---

## 17. Risk Assessment & Mitigation

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R1 | **Tenant isolation breach** — cross-tenant data leak | Low | Critical | Database-per-tenant; middleware enforced on every request; integration tests per tenant boundary; penetration testing |
| R2 | **Elastic Pool saturation** — noisy neighbor effect | Medium | High | DTU alerts at 80%; ability to move large tenants to dedicated DBs; per-tenant query timeout limits |
| R3 | **Azure AD B2C outage** — users cannot authenticate | Low | Critical | Implement token caching (short-lived offline); B2C SLA 99.9%; fallback to cached session |
| R4 | **Blob storage data loss** — document corruption or deletion | Low | Critical | Soft delete (30 days); blob versioning; geo-redundant storage (RA-GRS); daily backup verification |
| R5 | **SAS token abuse** — leaked download URL | Medium | Medium | 15-minute expiry; IP restriction on SAS (V2); download audit log; revocation via new storage key rotation |
| R6 | **Scope creep** — MVP expands beyond 20 weeks | High | High | Strict MVP feature freeze; V2 backlog is clearly separated; sprint reviews with stakeholders |
| R7 | **Performance degradation** — slow API under load | Medium | High | Load testing at M4; APM monitoring; database indexing strategy; Redis caching for hot paths |
| R8 | **Team attrition** — loss of key engineer | Medium | High | Documentation-first culture; pair programming; all infra in IaC; no single points of knowledge |
| R9 | **Regulatory non-compliance** — MENA data residency violation | Low | Critical | Azure region pinning (UAE North); no cross-region data flow; legal review at M1 |
| R10 | **Third-party dependency** — breaking change in Azure services | Low | Medium | Pin SDK versions; use LTS runtimes; test upgrades in staging; subscribe to Azure advisories |
| R11 | **Billing integration failure** — Stripe webhook desync | Medium | Medium | Idempotent webhook handlers; reconciliation job (daily); manual override in Admin Portal |
| R12 | **Virus in uploaded documents** — malware propagation | Low | High | Azure Defender for Storage; quarantine infected files; block download until scan completes |

---

## 18. Appendices

### A. Glossary

| Term | Definition |
|---|---|
| **Tenant** | A law firm subscribed to the platform, with its own isolated data and users |
| **Slug** | URL-safe short identifier for a tenant (e.g., `almansour`) |
| **Seat** | A licensed user slot; seats are consumed when users are marked active |
| **Case** | A legal matter or proceeding managed within the platform |
| **Court** | A judicial body configured by the tenant admin |
| **SAS Token** | Shared Access Signature — time-limited, scoped URL for Azure Blob access |
| **Elastic Pool** | Azure SQL feature allowing multiple databases to share compute resources |
| **B2C** | Azure AD Business-to-Consumer — identity platform for tenant users |
| **Entra ID** | Microsoft's enterprise identity platform (formerly Azure AD) for internal users |

### B. Wireframe Reference

All Tenant Portal wireframes are available as interactive HTML:
- **File:** `doc/LegalSaaS_Wireframes_v2.html`
- **Screens:** 13 complete screens covering all MVP features
- **Usage:** Open in browser; use screen selector dropdown to navigate; zoom controls available

### C. Key Configuration Values

| Setting | Dev | Staging | Production |
|---|---|---|---|
| **Azure Region** | UAE North | UAE North | UAE North |
| **DR Region** | — | — | West Europe |
| **SQL Elastic Pool DTU** | 50 eDTU | 100 eDTU | 200 eDTU |
| **Blob Redundancy** | LRS | GRS | RA-GRS |
| **Redis SKU** | C0 Basic | C1 Standard | C1 Standard |
| **App Service Plan** | B1 | S1 | P1v3 |
| **SAS Token TTL (download)** | 60 min | 15 min | 15 min |
| **SAS Token TTL (upload)** | 60 min | 30 min | 30 min |
| **Rate Limit (user)** | Unlimited | 100/min | 100/min |
| **Rate Limit (tenant)** | Unlimited | 1000/min | 1000/min |

### D. Approved File Types

| Extension | MIME Type | Max Size |
|---|---|---|
| `.pdf` | application/pdf | 50 MB |
| `.doc` | application/msword | 50 MB |
| `.docx` | application/vnd.openxmlformats-officedocument.wordprocessingml.document | 50 MB |
| `.xls` | application/vnd.ms-excel | 50 MB |
| `.xlsx` | application/vnd.openxmlformats-officedocument.spreadsheetml.sheet | 50 MB |
| `.jpg` / `.jpeg` | image/jpeg | 20 MB |
| `.png` | image/png | 20 MB |
| `.tiff` | image/tiff | 50 MB |

### E. Case Status State Machine

```
                ┌───────────┐
                │   Intake   │
                └─────┬─────┘
                      │
                ┌─────▼──────┐
                │ In Progress │◄────────────────┐
                └─────┬──────┘                  │
                      │                         │
                ┌─────▼─────┐            ┌──────┴──────┐
                │   Filed   │            │   Reopened  │
                └─────┬─────┘            └─────────────┘
                      │                         ▲
            ┌─────────▼─────────┐               │
            │ Awaiting Judgment │               │
            └─────────┬────────┘               │
                      │                         │
                ┌─────▼─────┐                  │
                │  Judgment  │──────────────────┘
                └─────┬─────┘
                      │
                ┌─────▼─────┐
                │  Closed   │
                └─────┬─────┘
                      │
                ┌─────▼──────┐
                │  Archived  │
                └────────────┘
```

**Transition Rules:**
- Intake → In Progress (lawyer begins work)
- In Progress → Filed (documents submitted to court)
- Filed → Awaiting Judgment (court accepted filing)
- Awaiting Judgment → Judgment (court issues decision)
- Judgment → Closed (matter resolved)
- Judgment → In Progress (case reopened / appeal)
- Closed → Archived (after retention period)
- Any active state → Closed (case withdrawn)

---

**END OF DOCUMENT**

*This document is the authoritative reference for LegalSaaS platform implementation. All architectural decisions are final. Deviations require written approval from the Technical Architect.*
