# LegalSaaS Monorepo - Initial Source Layout

This repository is bootstrapped to match the approved **LegalSaaS Architecture & Product Design Document** and its implementation constraints for runtime, identity, data isolation, and deployment.

## Repository structure

```text
.
├── backend/
│   └── src/
│       ├── TenantApi/
│       ├── AdminApi/
│       └── Shared/
│           ├── TenantResolution/
│           ├── Rbac/
│           └── DataAccess/
├── frontend/
│   └── apps/
│       ├── public-site/
│       ├── tenant-portal/
│       └── admin-portal/
└── infra/
    ├── bicep/
    └── k8s/
```

## Architectural alignment (source of truth)

- **Backend runtime:** .NET 8 Web API (`TenantApi`, `AdminApi`)
- **Frontend runtime:** React 18 + TypeScript + Vite (3 SPA apps)
- **Identity:** Azure AD B2C (tenant users) and Microsoft Entra ID / Azure AD (admin operators)
- **Data model:** Azure SQL database-per-tenant with a separate platform database
- **Tenant routing:** subdomain-based tenant resolution (`{tenant}.legalsaas.com`)
- **Deployment model:** Azure App Service for APIs, Azure Static Web Apps for SPAs
- **IaC and delivery:** Bicep + GitHub Actions

## Setup sequence

1. **Backend bootstrap**
   - Create .NET 8 solution under `backend/`.
   - Add projects for `TenantApi`, `AdminApi`, and shared libraries:
     - `Shared.TenantResolution`
     - `Shared.Rbac`
     - `Shared.DataAccess`
2. **Frontend bootstrap**
   - Initialize each app in `frontend/apps/` with React + TS + Vite:
     - `public-site`
     - `tenant-portal`
     - `admin-portal`
3. **Infrastructure bootstrap**
   - Implement Azure resources in `infra/bicep/` based on section 3.2 (Front Door, App Service, Static Web Apps, SQL, Blob, Redis, Key Vault, B2C, Service Bus, Monitor/App Insights).
   - Add local-cluster manifests and/or Helm overlays in `infra/k8s/` for development parity.
4. **CI/CD bootstrap**
   - Add GitHub Actions workflows for backend, frontend, and infra validation/deploy flows per environment.

## Environment matrix

| Dimension | dev | staging | prod |
|---|---|---|---|
| Purpose | Local/dev integration and rapid iteration | Pre-production validation and UAT | Production workloads |
| Azure region | UAE North | UAE North | UAE North |
| DR region | N/A | N/A | West Europe |
| API hosting | App Service Plan B1 | App Service Plan S1 | App Service Plan P1v3 |
| SPA hosting | Azure Static Web Apps (Standard) | Azure Static Web Apps (Standard) | Azure Static Web Apps (Standard) |
| SQL strategy | Platform DB + per-tenant DBs (elastic pool strategy) | Platform DB + per-tenant DBs | Platform DB + per-tenant DBs |
| Identity (tenant) | Azure AD B2C | Azure AD B2C | Azure AD B2C |
| Identity (admin) | Entra ID / Azure AD | Entra ID / Azure AD | Entra ID / Azure AD |
| IaC source | `infra/bicep/` | `infra/bicep/` | `infra/bicep/` |
| CI/CD | GitHub Actions | GitHub Actions + manual gates | GitHub Actions + manual gates |

## Notes

- This commit establishes the directory scaffold only.
- Implementation details must continue to follow `LegalSaaS_Architecture_and_Product_Design_Document.md`.
