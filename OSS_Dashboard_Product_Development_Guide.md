# OSS Dashboard Platform Product Development Guide

| Item | Details |
|---|---|
| Document title | OSS Dashboard Platform Product Development Guide |
| Document purpose | Definitive product architecture and delivery baseline |
| Primary audience | Product, Architecture, Backend, Frontend, DevOps, Security, IAM, QA and Operations |
| Classification | Internal / Confidential |
| Architectural position | TMF-compliant OSS integration, orchestration and operational dashboard platform |
| System-of-record model | External OSS/BSS platforms remain systems of record |

## 1. Purpose and Product Objective

The OSS Dashboard is a modular, metadata-driven product that provides a unified operational view across TMF-compliant OSS/BSS platforms. It supports inventory, ordering, alarms, catalog, customer, account, quote, event, administration and analytical capabilities through a single browser experience.

The product shall provide:

- TMF-compliant APIs based on official TM Forum OpenAPI specifications.
- Generated, typed API interfaces and domain models.
- Connections to multiple TMF-compliant external platforms in each supported domain.
- A React user interface backed by a Spring Boot Backend for Frontend (BFF).
- Browser authentication through the enterprise Keycloak login page.
- Server-side user sessions and server-side OAuth 2.0/OpenID Connect token handling.
- Metadata-driven search, list, details and form components.
- Explicit orchestration services for cross-domain capabilities.
- Central connection registration, capability validation and credential management.
- Fine-grained role-based authorization enforced by the backend.
- Auditing, correlation, resilience, observability and controlled operational workflows.
- Docker and Kubernetes deployment models.

The OSS Dashboard is not a system of record for service, resource, order, alarm, catalog, customer, account or quote information. External OSS/BSS systems remain authoritative. The platform reads from and writes to those systems through certified TMF connections.

### 1.1 Scope boundary

The product integrates only with external systems that satisfy a supported TMF contract and pass connection certification. It does not provide a general-purpose transformation layer for proprietary or non-TMF payloads.

The absence of a mapper does not mean that every vendor implementation is accepted automatically. A connection must declare and validate its TMF version, resources, operations, filters, pagination, authentication, extensions, errors and event capabilities before activation.

### 1.2 Product outcomes

The product shall enable authorized users to:

- Search and inspect operational data across one or more registered connections.
- Navigate relationships among services, resources, orders, alarms, customers and accounts.
- Use consolidated operational views such as Service 360 and Customer 360.
- Perform permitted changes with audit and approval controls.
- Monitor real-time events.
- Administer connections and installed TMF modules.
- Review platform activity and operational analytics.

## 2. Core Architectural Principles

1. **TMF contract first:** A supported module begins with the official TM Forum OpenAPI specification.
2. **Generated and typed:** Interfaces and models are generated during the build. Arbitrary OpenAPI documents are not interpreted to create runtime modules.
3. **External systems remain authoritative:** Operational domain records are not replicated into the platform database as master data.
4. **Connection, not custom code, per provider:** A certified provider using an already supported TMF contract is onboarded as a connection.
5. **Explicit orchestration:** Cross-API capabilities use dedicated orchestration modules rather than being placed inside a single TMF module.
6. **BFF security model:** Spring Boot owns OIDC browser login, tokens, application sessions and backend authorization. React remains token-free.
7. **Backend enforcement:** Frontend checks improve usability; backend checks provide security.
8. **Metadata-driven standard screens:** Common entities use reusable metadata-driven components; specialized operational views use explicit extensions.
9. **Configuration over source changes:** Environment values and connection records are configurable. Supporting a previously unsupported TMF API or contract version still requires generation, verification, build and deployment.
10. **Fail closed:** Invalid security, connection, approval and production configuration must not silently fall back to an insecure mode.
11. **Observable by default:** Requests propagate correlation identifiers and produce structured logs, metrics, traces and auditable security events.
12. **Modular monolith first:** Domain boundaries are explicit without introducing distributed deployment complexity prematurely.

## 3. Architecture Overview

```text
Browser
  |
  | Secure application-session cookie
  v
Spring Boot OSS BFF
  |-- OIDC login, callback and logout
  |-- Server-side session and token storage
  |-- Authorization and CSRF enforcement
  |-- BFF APIs and real-time bridge
  |-- Audit, correlation and configuration
  |
  +--> Metadata Engine
  +--> Connection Registry and Certification
  +--> TMF Modules
  |      |-- TMF638 Service Inventory
  |      |-- TMF639 Resource Inventory
  |      |-- TMF641 Service Ordering
  |      |-- TMF622 Product Ordering
  |      |-- TMF642 Alarm Management
  |      |-- TMF620 Product Catalog
  |      |-- TMF632 Party/Customer
  |      |-- TMF666 Account Management
  |      |-- TMF648 Quote Management
  |      `-- TMF688 Event Management
  |
  `--> Orchestration
         |-- Service 360
         |-- Customer 360
         |-- Global Search
         |-- Order Journey
         |-- Alarm Correlation
         |-- Analytics
         `-- Workflow Approval
                 |
                 v
       Certified TMF Connections
                 |
                 v
       External OSS/BSS Systems

Existing Enterprise Keycloak
  |-- Browser authentication
  |-- Enterprise SSO
  |-- Client roles and groups
  `-- OIDC endpoints
```

### 3.1 Work categories

#### Category A: Typed single-resource TMF operations

Standard search, read and supported write operations against one TMF resource and one selected connection. These use generated interfaces and models plus a thin typed gateway.

#### Category B: TMF operations with platform controls

Category A operations combined with connection selection, authorization, resilience, environment restrictions, auditing, credential resolution and capability checks.

#### Category C: Cross-API orchestration

Capabilities that call multiple modules or connections and compose results, including Service 360, Customer 360, global search, order journey, alarm correlation, analytics and workflow approval.

### 3.2 Build-time and runtime responsibilities

```text
Official TMF OpenAPI
  -> OpenAPI generation
  -> Generated interfaces and models
  -> Typed module gateway
  -> Validation and automated tests
  -> Application build
  -> Deployment
```

Runtime configuration can register a new connection for an installed module, enable or disable an installed module, and select routing or operational policy. Supporting a TMF API or contract version absent from the deployed artifact requires generation, test, build and deployment.

## 4. Identity and Access Management

### 4.1 Identity provider boundary

The organization’s existing Keycloak service provides authentication, enterprise SSO, groups and client roles. The OSS Dashboard does not install, operate, patch or administer Keycloak infrastructure and does not invoke Keycloak administration APIs from the product.

IAM administrators provision the realm or approved realm placement, the confidential client, roles, groups, mappings, redirect URIs and users according to enterprise policy.

### 4.2 Selected authentication architecture

The product uses a **Spring Boot BFF-managed OAuth 2.0/OpenID Connect Authorization Code Flow**.

Spring Boot acts as:

- The OIDC confidential client.
- The browser-login initiator.
- The Keycloak callback handler.
- The server-side token holder.
- The authenticated application-session owner.
- The BFF API for React.
- The final authorization-enforcement boundary.
- The initiator of local logout and, when configured, Keycloak logout.

React does not act as a confidential Keycloak client. React never receives or stores a client secret, access token, refresh token or ID token.

### 4.3 Browser redirect and authentication experience

```text
Browser requests a protected OSS Dashboard page
  -> BFF checks the server-side session
  -> BFF preserves the original requested URL
  -> Browser redirects to enterprise Keycloak
  -> User authenticates on the Keycloak-hosted page
  -> Keycloak redirects to the BFF callback
  -> BFF validates state, nonce, issuer, signature, audience and expiry
  -> BFF exchanges the authorization code
  -> Tokens remain server-side
  -> Session identifier is rotated
  -> Authenticated server-side session is created
  -> Browser receives a Secure, HttpOnly session cookie
  -> Browser returns to the originally requested page
```

The React application does not contain a username and password form. User credentials are entered only on the enterprise Keycloak page and must not pass through the OSS Dashboard.

### 4.4 SSO behavior

When the user has no compatible Keycloak SSO session:

```text
OSS Dashboard -> Keycloak login -> OSS Dashboard
```

When the user has a compatible active Keycloak SSO session:

```text
OSS Dashboard -> Keycloak -> immediate return to OSS Dashboard
```

The product does not copy or reuse another application’s session cookie. It relies on the enterprise Keycloak SSO session and maintains its own application session.

### 4.5 Keycloak client baseline

```yaml
client:
  client-id: oss-dashboard-client
  client-type: confidential
  client-authentication: enabled
  standard-flow: enabled
  direct-access-grants: disabled
  implicit-flow: disabled
  service-accounts: disabled
  full-scope-allowed: false
```

Each environment must use exact approved redirect and post-logout redirect URIs. Broad wildcard redirect patterns are prohibited.

Example callback pattern:

```text
https://<oss-dashboard-host>/login/oauth2/code/keycloak
```

OIDC discovery should use:

```text
https://<keycloak-host>/realms/<realm>/.well-known/openid-configuration
```

### 4.6 Browser, API and authorization behavior

- Protected browser page without a session: redirect to Keycloak.
- Successful login: redirect to the originally requested page.
- REST API request without an authenticated session: `401 Unauthorized`.
- Authenticated API request without a required role: `403 Forbidden`.
- Authorization failure: no new login redirect loop.
- Invalid or expired application session: API returns `401`; protected browser navigation begins authentication.
- Keycloak unavailable for login: controlled authentication-unavailable page with a correlation ID.

### 4.7 Session security

The BFF owns the application session.

```yaml
server:
  servlet:
    session:
      cookie:
        name: OSS_DASHBOARD_SESSION
        http-only: true
        secure: true
        same-site: lax
```

Production requirements:

- Secure and HttpOnly cookie attributes.
- Host-restricted cookie scope.
- Session ID rotation after login.
- Session fixation protection.
- Configured idle and absolute timeouts.
- Local server-side invalidation during logout.
- Shared session storage for horizontal scaling, or a documented affinity design.
- No OAuth/OIDC tokens in URLs, browser storage, JavaScript state, logs, API output or audit data.

### 4.8 CSRF protection

Cookie-authenticated state-changing operations require CSRF protection:

- POST
- PUT
- PATCH
- DELETE

CSRF protection must not be disabled globally. React obtains the approved CSRF token from the BFF and returns it using the configured request header. Any exclusion must be documented and security-reviewed.

### 4.9 Logout

The product supports an enterprise-configured policy:

- **Local logout:** invalidate only the OSS Dashboard server-side session.
- **Global SSO logout:** invalidate the local session and request logout at Keycloak.

Global SSO logout may affect other applications sharing the enterprise session, so the enterprise IAM owner selects the allowed default.

### 4.10 Groups and client roles

User-facing groups:

- `OssUser`
- `OssAdvancedUser`
- `OssAdministrator`

Domain and administration roles:

```text
service:read       service:write       service:delete
resource:read      resource:write      resource:delete
order:read         order:write
alarm:read         alarm:write
catalog:read
customer:read
account:read
quote:read         quote:write
event:read
analytics:read
admin:applications
admin:apis
admin:users
admin:audit
workflow:submit
workflow:view
workflow:approve
```

Recommended baseline mapping:

- `OssUser`: applicable domain read roles.
- `OssAdvancedUser`: applicable read and write roles plus `workflow:submit`.
- `OssAdministrator`: applicable read, write, delete and administration roles plus `workflow:view`.
- `workflow:approve`: assigned separately according to separation-of-duties policy.

Group membership does not have to grant every domain role. IAM policy may restrict users to selected domains.

### 4.11 Backend authorization

The BFF maps approved Keycloak client-role claims to Spring Security authorities. Every backend endpoint and orchestration action enforces its required authority. Frontend permission checks control navigation and presentation only.

### 4.12 Security enablement by environment

```yaml
security:
  authentication:
    enabled: true
  authorization:
    enabled: true
```

- Local/development may disable authentication or authorization for isolated work.
- UAT and Production require both controls.
- The application must refuse to start in UAT or Production if either control is disabled.
- Local insecure mode uses a clearly marked synthetic principal and visible development banner.
- A synthetic principal must not be loadable in a non-development profile.
- Production connections are unavailable while local insecure mode is active.

## 5. Supported TMF APIs

| TMF API | Capability | Domain |
|---|---|---|
| TMF638 | Service Inventory Management | Inventory |
| TMF639 | Resource Inventory Management | Inventory |
| TMF641 | Service Ordering Management | Orders |
| TMF622 | Product Ordering Management | Orders |
| TMF642 | Alarm Management | Alarms |
| TMF620 | Product Catalog Management | Catalog |
| TMF632 | Party Management | Customer |
| TMF666 | Account Management | Customer and Account |
| TMF648 | Quote Management | Quotes |
| TMF688 | Event Management | Real-time Events |

TMF639 is the resource-inventory contract. Product Catalog, Product Order and Product Inventory are distinct concepts:

- Product Catalog describes what can be offered or sold.
- Product Order describes what was requested.
- Product Inventory describes what a customer currently owns.

The baseline does not claim a Product Inventory capability. Product context in Service 360 and Customer 360 is derived from product-order references and available service characteristics.

## 6. TMF Generation and Module Strategy

Official TMF OpenAPI files are retained with version provenance. Generation is deterministic and performed by the build pipeline. Generated source is separate from handwritten source.

A module contains:

- Pinned OpenAPI specification.
- Generation configuration.
- Generated interfaces and models.
- Typed module gateway.
- Module-specific validation.
- Metadata.
- Unit, integration, contract and capability tests.

A previously unsupported TMF API or version requires:

1. Contract selection and licensing/provenance review.
2. Code generation.
3. Compilation and static validation.
4. Module implementation and configuration.
5. Contract, integration and security testing.
6. Build and deployment.

## 7. Connection Management and Certification

### 7.1 Connection registration

An administrator registers:

- Connection name and identifier.
- Supported TMF API.
- Base URL.
- Contract version.
- Environment classification.
- Authentication type and secret reference.
- Timeouts and approved resilience policy.
- Read/write enablement.
- Ownership and support contacts.

Credentials are never returned to the browser or exposed in ordinary administration APIs.

### 7.2 Connection states

```text
DRAFT
VALIDATING
ACTIVE
DEGRADED
DISABLED
REJECTED
```

Only an active, certified connection may receive normal traffic. Production writes require all applicable authorization and approval controls.

### 7.3 Capability profile

Each connection records:

- TMF API identifier and contract version.
- Supported resources and operations.
- Supported filters and sort behavior.
- Pagination behavior.
- PATCH support.
- Authentication type.
- Required extensions.
- Notification or event support.
- Error-format compatibility.
- Conformance result.
- Last validation time.
- Capability-profile version.

Revalidation is required after material endpoint, credential, contract or capability changes. The platform rejects incompatible traffic rather than translating nonconformant payloads.

### 7.4 Connector design

Common HTTP transport is shared, while domain gateways remain typed.

```text
tmf-core/http/
|-- TmfHttpExecutor
|-- AuthenticationInterceptor
|-- CorrelationInterceptor
|-- PaginationExtractor
|-- ErrorDecoder
`-- RequestContext

tmf638-service-inventory/
|-- generated/api/
|-- generated/model/
`-- gateway/ServiceInventoryGateway

tmf639-resource-inventory/
|-- generated/api/
|-- generated/model/
`-- gateway/ResourceInventoryGateway
```

The shared transport handles authentication, correlation, timeout, telemetry and error decoding. Typed gateways preserve module-specific resources, payload variants, pagination, conditional requests, notifications and non-CRUD operations.

### 7.5 Multi-connection execution

A selected-connection request targets one certified connection. An `ALL` request fans out only to eligible active connections, applying concurrency limits, authorization, cancellation, source attribution and per-connection timeouts.

### 7.6 Global search response

```json
{
  "items": [],
  "partial": true,
  "sources": [
    {
      "connectionId": "inventory-prod-1",
      "status": "SUCCESS",
      "durationMs": 280,
      "itemCount": 22
    },
    {
      "connectionId": "inventory-prod-2",
      "status": "TIMEOUT",
      "durationMs": 5000,
      "itemCount": 0
    }
  ],
  "page": {
    "limit": 50,
    "nextCursor": "opaque-cursor"
  },
  "correlationId": "correlation-id"
}
```

Global search defines:

- Opaque platform cursor pagination.
- Stable platform-level sorting.
- Domain-specific identity and deduplication rules.
- Per-source status and elapsed time.
- Partial success rather than silent omission.
- Maximum participating connections and merged records.
- Cancellation when the caller disconnects.
- No leakage of results from unauthorized connections.

## 8. Orchestration Architecture

```text
orchestration/
|-- orchestration-core/
|   |-- aggregation/
|   |-- partialfailure/
|   |-- source/
|   |-- concurrency/
|   |-- timeout/
|   |-- authorization/
|   `-- correlation/
|-- service360/
|-- customer360/
|-- globalSearch/
|-- orderJourney/
|-- alarmCorrelation/
|-- analytics/
`-- workflowApproval/
```

The orchestration core standardizes concurrency, time budgets, partial failure, source attribution, aggregation, deduplication, authorization, cancellation and correlation propagation.

### 8.1 Service Details composition

UC-02 is a composite BFF experience, not one large TMF638 response. The UI loads:

```text
GET service core
GET related resources
GET related service orders
GET active alarms
```

The core service panel renders first. Relationship panels load independently, preserve a common correlation ID and display partial failures without hiding successful data. These relationship queries are exposed through the `service360` orchestration boundary.

### 8.2 Service 360 data model

```text
Customer
  -> Account
  -> Product Order context
  -> Service Order
  -> Service
  -> Resource
  -> Alarm
```

Service 360 must label product-order context accurately and must not represent it as customer-owned Product Inventory.

### 8.3 Workflow approval

Production change workflow:

```text
Requester submits action
  -> immutable approval request
  -> authorized approver approves or rejects
  -> approved action executes against the certified connection
  -> approval and execution outcomes are audited
```

Rules:

- Requesters cannot approve their own requests.
- Approval expires after a configured period.
- Approved payloads are immutable.
- A payload change creates a new request.
- Rejected or expired requests cannot execute.
- Failure to verify approval blocks execution.
- Approval and execution are separate audit events.

## 9. Metadata and Frontend UI Engine

### 9.1 Metadata lifecycle

```text
Official TMF OpenAPI
  -> Generated baseline UI metadata
  -> Product override metadata
  -> Environment display settings
  -> Validated metadata snapshot
```

Metadata defines:

- Entity and metadata schema versions.
- Compatible TMF contract versions.
- Fields, types, required flags and enumerations.
- Labels, layout, search, sorting and visibility.
- References, relationships and characteristics.
- Required permissions per operation or field.
- Sensitive-field masking.
- Localization resources.
- Override precedence and provenance.

Metadata is validated during build and startup. Unknown fields and incompatible versions fail according to an explicit validation policy. Activated snapshots are versioned and auditable.

### 9.2 UI components

```text
ui-engine/
|-- GenericTable/
|-- GenericForm/
|-- GenericDetails/
|-- GenericSearch/
|-- GenericFilter/
|-- GenericField/
|-- GenericReference/
|-- GenericRelationship/
`-- GenericCharacteristic/
```

Specialized extensions are used for topology, workflow visualization, dashboards and other capabilities that do not reduce to standard metadata-driven forms and tables.

## 10. Configuration Strategy

Environment configuration is externalized. Connections are runtime records in the Connection Registry, not YAML entries.

```text
config/
|-- application.yaml
|-- auth.yaml
|-- database.yaml
`-- modules.yaml
```

### 10.1 Terminology

- **Rebuild:** compile and package a new artifact.
- **Redeploy:** replace the running artifact.
- **Restart:** restart the same artifact with changed configuration.
- **Hot reload:** apply configuration without a restart.

Enabling or disabling an installed TMF module does not require a rebuild. In the initial operating model, a `modules.yaml` change takes effect after restarting the same artifact. Runtime module activation may be implemented through the Module Registry later.

### 10.2 Authentication configuration

```yaml
security:
  authentication:
    enabled: true
  authorization:
    enabled: true
  oidc:
    registration-id: keycloak
    issuer-uri: https://<keycloak-host>/realms/<realm>
    client-id: oss-dashboard-client
    client-secret: ${OSS_KEYCLOAK_CLIENT_SECRET}
    scopes:
      - openid
      - profile
      - email
  session:
    idle-timeout: 30m
    absolute-timeout: 8h
    cookie-name: OSS_DASHBOARD_SESSION
    secure: true
    http-only: true
    same-site: Lax
  logout:
    mode: enterprise-approved
    local-logout-path: /logout
    post-logout-redirect-uri: https://<oss-dashboard-host>/
```

### 10.3 Database configuration

```yaml
database:
  mode: existing
  existing:
    host: <database-host>
    port: 5432
    name: oss_dashboard
    username: oss_app
    password: ${OSS_DB_PASSWORD}
    ssl: true
```

Enterprise environments use a database provisioned under enterprise DBA controls. Local automation may provide a disposable development database.

### 10.4 Module configuration

```yaml
tmf:
  modules:
    tmf638:
      enabled: true
    tmf639:
      enabled: true
    tmf641:
      enabled: true
    tmf642:
      enabled: true
    tmf622:
      enabled: false
```

The deployed artifact determines supported APIs and versions. Configuration determines which installed modules are active.

## 11. Secrets and Credentials

- Keycloak client secrets, database passwords and connection credentials are never committed as plaintext.
- Secret values are resolved from an approved external secrets manager or controlled environment injection.
- Connection credentials use envelope encryption when retained in the platform database.
- Decryption occurs in memory only for the outbound operation.
- APIs return masked credential metadata, never secret values.
- Logs, traces, metrics and audit events apply secret redaction.
- Rotation procedures identify owner, cadence, overlap and rollback behavior.

## 12. Platform Persistence

The platform database stores only platform-owned state:

```text
OSS Platform Database
|-- User preferences
|-- Saved searches
|-- Dashboard customizations
|-- Audit events
|-- Connection registry
|-- Connection capability profiles
|-- Installed-module registry state
|-- Workflow and approval state
|-- Metadata snapshot references
`-- Job and integration status
```

It does not store authoritative copies of external service, resource, order, alarm, customer or catalog inventories.

## 13. Backend Repository Structure

```text
oss-dashboard-backend/
|-- oss-app/
|   |-- src/main/java/.../ossapp/
|   |   |-- gateway/
|   |   |-- security/
|   |   |   |-- oidc/
|   |   |   |-- login/
|   |   |   |-- session/
|   |   |   |-- logout/
|   |   |   |-- authorization/
|   |   |   |-- csrf/
|   |   |   `-- claims/
|   |   |-- audit/
|   |   |-- realtime/
|   |   `-- config/
|   `-- src/test/
|-- tmf/
|   |-- tmf-core/
|   |   |-- http/
|   |   |-- error/
|   |   |-- pagination/
|   |   |-- filtering/
|   |   |-- validation/
|   |   |-- logging/
|   |   |-- correlation/
|   |   |-- security/
|   |   `-- resilience/
|   |-- metadata-engine/
|   |-- connection-registry/
|   |-- tmf638-service-inventory/
|   |-- tmf639-resource-inventory/
|   |-- tmf641-service-ordering/
|   |-- tmf622-product-ordering/
|   |-- tmf642-alarm-management/
|   |-- tmf620-product-catalog/
|   |-- tmf632-party-customer/
|   |-- tmf666-account-management/
|   |-- tmf648-quote-management/
|   |-- tmf688-event-management/
|   `-- orchestration/
|       |-- orchestration-core/
|       |-- service360/
|       |-- customer360/
|       |-- globalSearch/
|       |-- orderJourney/
|       |-- alarmCorrelation/
|       |-- analytics/
|       `-- workflowApproval/
|-- config/
|-- others/
|   |-- docker/
|   |-- scripts/
|   |-- docs/
|   |   |-- keycloak-realm-client-setup.md
|   |   `-- environment-setup-checklist.md
|   `-- ci/
`-- pom.xml
```

## 14. Frontend Repository Structure

```text
oss-dashboard-frontend/
|-- ui-engine/
|-- extensions/
|   |-- ServiceTopology/
|   |-- AlarmDashboard/
|   |-- OrderWorkflow/
|   `-- ResourceVisualization/
|-- metadata/
|-- screens/
|   |-- ServiceSearch/
|   |-- ServiceDetails/
|   |-- ResourceSearch/
|   |-- ResourceDetails/
|   |-- OrderSearch/
|   |-- OrderDetails/
|   |-- AlarmSearch/
|   |-- CustomerSearch/
|   |-- AccountManagement/
|   |-- CatalogBrowser/
|   |-- QuoteManagement/
|   |-- Service360/
|   |-- Customer360/
|   |-- Analytics/
|   |-- EventsMonitor/
|   `-- Administration/
|-- auth/
|   |-- AuthenticatedUserProvider/
|   |-- SessionStatus/
|   |-- LoginRedirect/
|   |-- Logout/
|   |-- PermissionGuard/
|   `-- UnauthorizedView/
|-- realtime/
|-- navigation/
|-- api-client/
`-- shared/
```

The frontend has no token persistence, refresh-token handling, client-secret handling or local login form.

## 15. Use Case Catalog

### 15.1 Inventory

| ID | Use case | Scope | APIs | Roles |
|---|---|---|---|---|
| UC-01 | Service Search | Search by service ID, name, interaction reference or supported customer reference | TMF638 | service:read |
| UC-02 | Service Details | Core details plus independently loaded resources, service orders and alarms | TMF638, TMF639, TMF641, TMF642 | service:read and applicable related read roles |
| UC-03 | Resource Search | Search logical, physical, telephone and IP resources | TMF639 | resource:read |
| UC-04 | Resource Details | Specification, characteristics, interfaces, addresses and relationships | TMF639, TMF638 | resource:read and service:read |

### 15.2 Orders

| ID | Use case | Scope | APIs | Roles |
|---|---|---|---|---|
| UC-05 | Order Search | Search Product Orders and Service Orders | TMF622, TMF641 | order:read |
| UC-06 | Order Journey Trace | Product Order to Service Order to Service to Resource | TMF622, TMF641, TMF638, TMF639 | order:read, service:read, resource:read |
| UC-07 | Order Details | Tasks, milestones, state, links and fulfillment information | TMF641 | order:read |

### 15.3 Alarms

| ID | Use case | Scope | APIs | Roles |
|---|---|---|---|---|
| UC-08 | Alarm Search | Search by alarm ID, severity, resource or service | TMF642 | alarm:read |
| UC-09 | Service Impact View | Active alarms related to a service | TMF638, TMF642 | service:read, alarm:read |
| UC-10 | Alarm Dashboard | Counts by severity, connection, service and resource | TMF642 | alarm:read |
| UC-11 | Alarm Correlation | Alarm to service and resource relationships | TMF642, TMF638, TMF639 | alarm:read, service:read, resource:read |

### 15.4 Catalog, Customer, Account and Quotes

| ID | Use case | Scope | APIs | Roles |
|---|---|---|---|---|
| UC-12 | Product Catalog Browsing | Catalogs, offerings and specifications | TMF620 | catalog:read |
| UC-13 | Service Specification Lookup | Product offering and service-specification context | TMF620, TMF638 | catalog:read, service:read |
| UC-14 | Customer Search | Search party/customer records | TMF632 | customer:read |
| UC-15 | Account Details | Billing, service and customer accounts | TMF666 | account:read |
| UC-16 | Quote Search | Search by quote ID, customer and status | TMF648 | quote:read |
| UC-17 | Quote Lifecycle and CRUD | Create, change and manage supported quote transitions | TMF648 | quote:read, quote:write |

### 15.5 Controlled changes

| ID | Use case | Scope | APIs | Roles |
|---|---|---|---|---|
| UC-18 | Service change operations | Create, change, deactivate or delete when supported and approved | TMF638 | service:write, service:delete |
| UC-19 | Resource change operations | Create, change or delete when supported and approved | TMF639 | resource:write, resource:delete |

Writes must respect capability profiles, environment policy, optimistic locking, idempotency policy, audit requirements and workflow approval rules.

### 15.6 Multi-connection and orchestration

| ID | Use case | Scope | Roles |
|---|---|---|---|
| UC-20 | Selected Connection Search | Search one eligible certified connection | Applicable domain read role |
| UC-21 | Global Search | Search eligible active connections with partial-result metadata | Applicable domain read roles |
| UC-22 | Service 360 | Core and complete cross-domain capabilities included in the Current Development Roadmap | Applicable cross-domain read roles |
| UC-23 | Customer 360 | Accounts, product-order context, services and alarms | Applicable cross-domain read roles |
| UC-24 | Relationship Explorer | Graph view across order, service, resource and alarm relationships | Applicable cross-domain read roles |
| UC-25 | Real-Time Event Monitoring | Live service, order, alarm and inventory events | event:read |

### 15.7 Administration and analytics

| ID | Use case | Scope | Roles |
|---|---|---|---|
| UC-26 | Application Registry | Register, certify, change, disable and review connections | admin:applications |
| UC-27 | API / Module Registry | Configure installed modules and their connections | admin:apis |
| UC-28 | Role and Group Management | Performed in enterprise Keycloak | Enterprise IAM responsibility |
| UC-29 | Audit Dashboard | Review searches, changes, approvals and administration | admin:audit |
| UC-30 | Workflow Approval | Submit, view, approve or reject controlled operations | workflow:submit, workflow:view, workflow:approve |
| UC-31 | Dashboard Analytics | Operational counts and connection metrics | analytics:read |

UC-27 does not create support for a previously unsupported API at runtime. Such support requires generation, validation, testing, build and deployment but does not require a redesign of the shared core.

## 16. Development Roadmap

### Current Development Roadmap

```text
Service Search
  -> Service Details
  -> Resource Search
  -> Order Search
  -> Alarm Search
  -> Service 360 Core
```

Included:
- UC-01, UC-02, UC-03, UC-04, UC-05, UC-06, UC-07, UC-08, UC-09, UC-10 and UC-11.
- UC-12 through UC-19.
- UC-20 selected-connection search.
- UC-21 Global Search.
- UC-22 Service 360 Core and Service 360 Complete.
- UC-23 through UC-25.
- UC-26 initial connection registration and certification.
- UC-27 configuration of installed modules.
- UC-28 enterprise Keycloak client, roles, groups and mappings.
- UC-29 through UC-31.
- BFF authentication, sessions, CSRF and backend authorization.
- Expanded connection maintenance and revalidation.

Service 360 Core covers core service details, related resources, related service orders and active alarms using TMF638, TMF639, TMF641 and TMF642.

Service 360 Complete adds customer, account, product-order and broader cross-domain context. It does not claim Product Inventory unless a dedicated supported product-inventory contract is introduced.

### Future Development Roadmap

Potential capabilities include AI-assisted correlation, impact analysis, anomaly detection, advanced topology, bulk operations and broader federated discovery. Each capability requires a separate product, data-governance and safety assessment before commitment.
## 17. Testing Strategy

### 17.1 Core engineering tests

- Unit tests for services, gateways, validation and business rules.
- Generated-contract verification.
- Module integration tests against TMF-compliant mocks.
- Registry and credential-encryption tests.
- Metadata schema, compatibility and rendering tests.
- BFF API and frontend component tests.
- End-to-end browser-to-BFF-to-module-to-mock tests.

### 17.2 Authentication and session tests

- Protected browser access redirects to Keycloak.
- Successful login returns to the original requested URL.
- Compatible SSO avoids unnecessary credential entry.
- Invalid state and nonce are rejected.
- Expired authorization codes are rejected.
- Invalid issuer, signature, audience and expiry are rejected.
- Unauthenticated API calls return 401.
- Missing roles return 403.
- Authorization failures do not create login loops.
- Tokens and client secrets never reach React.
- Session cookies are Secure and HttpOnly.
- Session fixation is prevented.
- CSRF failures reject protected writes.
- Logout invalidates the local session.
- Global logout follows enterprise policy.
- Disabled users cannot create a new session.
- UAT and Production startup fails when security controls are disabled.

### 17.3 Connection compatibility tests

- Contract version and required resources.
- Supported operations, filters and sorting.
- Pagination and PATCH behavior.
- Extensions and error responses.
- Authentication and credential rotation.
- Event or notification support.
- Revalidation and state transitions.

### 17.4 Resilience and orchestration tests

- Timeout, circuit breaker and bulkhead behavior.
- Partial failure and source attribution.
- Retry safety and write retry restrictions.
- Slow and recovered connections.
- Caller cancellation.
- Concurrency and maximum fan-out limits.
- Unauthorized-source filtering.
- Stable pagination and deduplication.

### 17.5 Workflow and write-security tests

- Production write without approval is rejected.
- A requester cannot self-approve.
- Expired or rejected approval cannot execute.
- Edited payload requires a new approval.
- Approved payload immutability is enforced.
- ETag and optimistic locking prevent lost changes.
- Idempotency prevents duplicate supported writes.
- Audit events link request, approval and execution.
- Secrets are absent from logs, traces and API output.

CI must not depend on live production external systems.

## 18. Deployment and Operations

The product begins as a Spring Boot modular monolith with React served through or alongside the BFF under the approved ingress topology.

```text
Developer environment
  -> OSS application
  -> TMF mocks
  -> local disposable database when required
  -> enterprise Keycloak or approved developer-only Keycloak

Enterprise environment
  -> Kubernetes Ingress
  -> OSS application replicas
  -> shared server-side session repository
  -> enterprise database
  -> enterprise secrets manager
  -> enterprise Keycloak
  -> certified external TMF systems
```

Operational requirements:

- TLS at all external boundaries.
- Correct forwarded-header and external-base-URL configuration.
- Health, readiness and liveness probes.
- Structured logs and correlation identifiers.
- Metrics and distributed tracing.
- Central secret injection and rotation.
- Database backup and recovery.
- Session-store availability and cleanup.
- Controlled deployment and rollback.
- Security configuration validation before traffic acceptance.

Individual TMF modules remain logical modules within the application unless operational evidence justifies independent deployment.

## 19. Enterprise Onboarding

The environment setup checklist shall include:

1. Confirm platform, IAM, DBA, security and external-system owners.
2. Provision the enterprise database and schema.
3. Provision secrets and rotation ownership.
4. Select or provision the Keycloak realm placement.
5. Create the confidential BFF client.
6. Configure exact redirect, origin and logout URIs.
7. Create roles, groups and mappings.
8. Configure issuer discovery and client-secret references.
9. Configure the shared session design.
10. Register external TMF connections.
11. Run connection certification.
12. Enable installed modules.
13. Execute security, contract, resilience and end-to-end verification.
14. Record operational ownership and support procedures.

`others/docs/keycloak-realm-client-setup.md` shall document client setup, roles, groups, user provisioning, URI configuration, secret handoff, SSO, logout, session policy, key rotation and troubleshooting.

## 20. Production Readiness Controls

Foundation implementation may begin with:

- Repository and modular-monolith skeleton.
- OpenAPI generation pipeline.
- TMF shared HTTP core.
- TMF638 and TMF639 read-only modules.
- Connection Registry and certification proof of concept.
- Metadata engine proof of concept.
- Mock TMF endpoints.
- BFF Keycloak integration.
- React shell and session-aware navigation.
- Audit, correlation and observability foundations.

Before production write operations are enabled, define and verify:

- ETag and optimistic-locking rules.
- Idempotency behavior.
- Permitted retry conditions for writes.
- Delete versus deactivate policy.
- Audit-event schema and retention.
- Approval expiry and payload immutability.
- Connection-specific write capabilities.
- Production security fail-fast behavior.
- Prevention of configuration-based approval or IAM bypass.

## 21. Architecture Decisions Summary

1. Spring Boot BFF-managed OIDC Authorization Code Flow is the browser authentication model.
2. React is token-free and uses only the secure BFF application session.
3. Generated TMF contracts are build-time dependencies.
4. Runtime configuration manages installed modules and certified connections.
5. Category C orchestration is separated from typed TMF modules.
6. Service 360 Core is delivered before Service 360 Complete.
7. Product-order context is not described as Product Inventory.
8. Workflow approval is separately assignable from administration.
9. Connection certification operationalizes the TMF-compliant-only boundary.
10. Production security failures stop startup or block the affected operation.

## 22. Enterprise Decisions to Record

The following environment-specific decisions must be recorded before production deployment:

- Keycloak host, realm placement and IAM owner.
- Environment hostnames and exact redirect URIs.
- Client-secret rotation process.
- Idle and absolute session timeout values.
- Local or global SSO logout policy.
- Shared session repository or approved affinity design.
- Exact domain-role assignments.
- Workflow approver assignment policy.
- External secrets-manager integration.
- Supported TMF contract versions per module.
- Connection certification evidence and owners.

## 23. Validation Checklist

The product baseline is internally consistent when all answers below are **Yes**:

- Is Spring Boot the sole OIDC confidential client for browser login?
- Is React free of client secrets and OAuth/OIDC tokens?
- Do unauthenticated API requests return 401 and unauthorized requests return 403?
- Are CSRF controls present for cookie-authenticated writes?
- Can UAT or Production refuse startup when security is disabled?
- Are unsupported TMF APIs treated as build-time work rather than runtime registration?
- Are installed modules distinguished from registered connections?
- Is Service 360 Core aligned with the Current Development Roadmap dependencies?
- Is Service 360 Complete aligned with the Current Development Roadmap domains?
- Are Product Catalog, Product Order and Product Inventory distinguished?
- Is connection certification required before activation?
- Are workflow approvers assigned independently from administrators?
- Do global-search responses expose partial failures and source attribution?
- Are metadata versions, validation and override precedence defined?
- Are secrets prohibited from browser storage, logs and API output?
- Are production writes protected by capability, authorization, concurrency, audit and approval controls?

---

**Document status:** Definitive product architecture and delivery baseline.
