# Proposal: Table-Backed Authorization Using Cedar Policies

**Scope:** Centralized authorization for application *actions*, *pages*, and *APIs*, with all authorization metadata, policies, and grants persisted in relational tables and evaluated by the Cedar policy engine.

**Status:** Draft for review
**Date:** August 2026

---

## 1. Executive Summary

Today authorization logic is typically scattered across controller annotations, UI conditionals, and ad-hoc `role_permission` tables. This proposal replaces that with a single authorization service built on **Cedar**, an open-source policy language and evaluation engine from AWS.

The core idea: **the database stores the catalog, the grants, and the policy text; Cedar owns the decision logic.**

Three enforcement surfaces are covered:

| Surface | Question asked | Where enforced |
|---|---|---|
| **Page** | Can this user open `/billing/invoices`? Should this menu item render? | UI shell + BFF |
| **API** | Can this user call `POST /api/v1/invoices/{id}/approve`? | Gateway / Spring filter |
| **Action** | Can this user approve *invoice 4711*, given its amount and owner? | Service layer, after the object is loaded |

All three resolve to the same primitive: a Cedar request of the shape `(principal, action, resource, context)` evaluated against a versioned policy set.

**Why Cedar rather than a hand-rolled permission matrix or OPA/Rego:** Cedar policies never authorize by default — access must be explicitly permitted — and the engine returns the decision *plus the list of determining policies* that produced it, which gives us auditability for free. The language covers RBAC, ABAC, and ReBAC in one vocabulary, so we can start with plain role checks and add attribute conditions later without changing the enforcement plumbing. The Cedar team modeled the language in Lean, proved properties about its design, implemented it in Rust, and benchmarked it as substantially faster than OpenFGA and Rego. A first-party Java binding is published to Maven Central (`com.cedarpolicy:cedar-java`, currently 4.10.0), and there is an optional managed path via Amazon Verified Permissions if we later want to offload hosting.

---

## 2. Goals and Non-Goals

### Goals

- **G1.** One authoritative store for what pages, APIs, and actions exist and who may reach them.
- **G2.** Authorization changes deployable without an application release.
- **G3.** Every decision explainable — "denied by policy `F-901-mfa-required`" — and auditable.
- **G4.** UI and API enforcement derived from the *same* grants, so the menu never shows something the API will reject, and vice versa.
- **G5.** Default-deny posture with organizational guardrails (`forbid`) that no delegated grant can override.
- **G6.** Sub-millisecond in-process decisions on the API gateway hot path.

### Non-Goals

- **Authentication.** Identity, MFA, and session management stay with the existing IdP. We consume claims.
- **User/group mastering.** Group membership is sourced from the IdP where one exists; the authorization store mirrors it for evaluation.
- **Row-level data filtering.** Cedar answers yes/no on a *known* resource. Turning policies into SQL predicates for list queries is out of scope for Phase 1 (see §11).

---

## 3. Key Design Decisions

### D1. Tables hold the catalog and the grants; Cedar holds the logic

The most common failure mode when adopting a policy engine is keeping a legacy `role_permission` matrix *and* writing policies — producing two sources of truth that silently diverge. We avoid this by assigning each store a distinct job:

| Concern | Lives in | Rationale |
|---|---|---|
| What pages / APIs / actions exist | Tables (`auth_page`, `auth_api`, `auth_action`) | Inventory data; needs joins, reports, referential integrity |
| Which action an API or page requires | Tables (FK on the catalog row) | Static routing metadata |
| Who is in which group / role | Tables (`auth_principal_parent`), mirrored from IdP | Membership data |
| Attributes of users and resources | Tables (`auth_*_attribute`) | Feeds Cedar entity attributes at request time |
| Conditional logic (MFA, tenancy, thresholds, time of day) | **Cedar policy text**, stored in `auth_policy` | This is the part that benefits from a real language |
| Individual grants ("Alice is approver for UK") | **Template-link rows** (`auth_policy_template_link`) | Still a table, still queryable, but evaluated by Cedar |

### D2. Grants are modeled as template-linked policies, not as a permission matrix

Cedar supports policy templates with `?principal` / `?resource` placeholders. Template-linked policies stay linked to their template, so editing the template statement immediately changes the behavior of every linked policy for all subsequent decisions. That is exactly the semantics we want for role definitions: edit the "Approver" template once, and every approver assignment updates.

Cedar's own guidance is that the preferred RBAC approach is principal *groups* as roles, backed by an IdP; where that is not available, role assignment can be managed inside the policy store using one template per role, instantiated per user and resource scope.

We use **both**, split by scope:

- **Global roles** (`billing-analyst`, `admin`) → IdP groups → `auth_principal_parent` rows → matched by static policies using `principal in Group::"..."`.
- **Scoped grants** ("Alice approves for the UK entity only") → `auth_policy_template_link` rows → template-linked policies.

This keeps the policy set small — roles do not multiply per user — while making scoped delegation a simple row insert with an `expires_at` column.

### D3. Two enforcement granularities, one engine

- **Coarse / entry-point check** — the resource is the `Page` or `Api` entity itself. Needs no business data, so it runs in the gateway filter before any service is invoked. Purely catalog-driven.
- **Fine / object check** — the resource is the domain entity (`Invoice::"4711"`). Runs in the service after the object is loaded, when its attributes (owner, amount, state) are available.

Both use the same policy set and the same `isAuthorized` call. The coarse check is a cheap early rejection; the fine check is the real security boundary.

### D4. Policy sets are published as immutable, hashed versions

Editing a policy row does not change runtime behavior. A *publish* operation snapshots the active policies into `auth_policy_set_version` with a content hash. Every PDP node reports which version it has loaded, so we can detect drift, roll back atomically, and stamp the version into every decision log record.

---

## 4. Authorization Model (Cedar Schema)

The Cedar schema is the contract between the tables and the policies. It is generated from the catalog tables, stored in `auth_cedar_schema`, and used by Cedar's validator to reject policies that reference non-existent actions or attributes.

```cedar
namespace BillingPortal {

  // ---------- Principals ----------
  entity Group;

  entity User in [Group] = {
    tenantId:     String,
    department:   String,
    employeeType: String,
    costCenter?:  String,
  };

  // ---------- Containers (targets of the `in` operator) ----------
  entity Module;     // groups Pages:  "billing", "admin"
  entity ApiGroup;   // groups Apis:   "invoice-write", "reporting-read"

  // ---------- Enforcement-surface resources ----------
  entity Page in [Module] = {
    pageKey:        String,
    tenantId:       String,
    classification: String,   // PUBLIC | INTERNAL | RESTRICTED
  };

  entity Api in [ApiGroup] = {
    apiKey:         String,
    httpMethod:     String,
    tenantId:       String,
    classification: String,
  };

  entity Feature in [Page] = {   // buttons, tabs, grid columns
    featureKey:     String,
    classification: String,
  };

  // ---------- Domain resources (fine-grained checks) ----------
  entity Invoice = {
    tenantId: String,
    owner:    User,
    amount:   Long,
    state:    String,
  };

  // ---------- Action groups ----------
  action ReadActions;
  action WriteActions;
  action ApproverActions in [WriteActions];

  // ---------- Actions ----------
  action "viewPage" in [ReadActions]
    appliesTo {
      principal: [User],
      resource:  [Page],
      context:   { mfa: Bool, srcIp: ipaddr, tenantId: String }
    };

  action "useFeature" in [ReadActions]
    appliesTo {
      principal: [User],
      resource:  [Feature],
      context:   { mfa: Bool, srcIp: ipaddr, tenantId: String }
    };

  action "callApi" in [ReadActions]
    appliesTo {
      principal: [User],
      resource:  [Api],
      context:   { mfa: Bool, srcIp: ipaddr, tenantId: String }
    };

  action "viewInvoice" in [ReadActions]
    appliesTo { principal: [User], resource: [Invoice],
                context: { mfa: Bool, srcIp: ipaddr, tenantId: String } };

  action "approveInvoice" in [ApproverActions]
    appliesTo { principal: [User], resource: [Invoice],
                context: { mfa: Bool, srcIp: ipaddr, tenantId: String } };
}
```

`callApi` is deliberately generic: an endpoint's authorization intent is carried by which `ApiGroup` it belongs to and by its `classification` attribute, both of which come from the catalog tables. Endpoints needing semantic differentiation get a business action (`approveInvoice`) attached via `auth_api.required_action_id`, plus a fine-grained check in the service.

### Sample policies

```cedar
// P-001  Any authenticated user may open public pages.
@id("P-001-public-pages")
permit (principal, action == BillingPortal::Action::"viewPage", resource)
when { resource.classification == "PUBLIC" };

// P-010  Role grant: analysts get read access across the billing module.
@id("P-010-analyst-billing-read")
permit (
  principal in BillingPortal::Group::"billing-analyst",
  action in BillingPortal::Action::"ReadActions",
  resource in BillingPortal::Module::"billing"
);

// P-011  Same role, the API side of the same permission.
@id("P-011-analyst-billing-api-read")
permit (
  principal in BillingPortal::Group::"billing-analyst",
  action in BillingPortal::Action::"ReadActions",
  resource in BillingPortal::ApiGroup::"billing-read"
);

// T-020  TEMPLATE: scoped approver role, instantiated per assignment.
@id("T-020-approver")
permit (
  principal == ?principal,
  action in BillingPortal::Action::"ApproverActions",
  resource in ?resource
);

// P-030  Self-service: users can always view their own invoices.
@id("P-030-own-invoice")
permit (
  principal,
  action == BillingPortal::Action::"viewInvoice",
  resource
) when { resource.owner == principal };

// F-900  GUARDRAIL: no cross-tenant access, ever.
@id("F-900-tenant-isolation")
forbid (principal, action, resource)
unless {
  resource has tenantId && principal has tenantId &&
  resource.tenantId == principal.tenantId
};

// F-901  GUARDRAIL: restricted resources require MFA on this session.
@id("F-901-mfa-for-restricted")
forbid (principal, action, resource)
when {
  resource has classification &&
  resource.classification == "RESTRICTED" &&
  context.mfa == false
};

// F-902  GUARDRAIL: approvals above threshold need a second control.
@id("F-902-approval-ceiling")
forbid (
  principal,
  action == BillingPortal::Action::"approveInvoice",
  resource
) when {
  resource has amount && resource.amount > 250000 &&
  !(principal in BillingPortal::Group::"senior-approver")
};
```

> **Safety note on `forbid` policies.** Cedar *skips on error*: if a policy's evaluation raises an error, it is dropped from the decision entirely. A guardrail `forbid` that errors — for example because an expected attribute is missing on the entity — is therefore silently skipped, which fails **open** for that policy. Every `forbid` in our set must guard attribute access with `has`, as shown above, and CI must reject any `forbid` containing an unguarded attribute dereference. This is the single most important correctness rule in the whole design.

---

## 5. Data Model

### 5.1 Overview

```
auth_application ─┬─ auth_cedar_schema
                  ├─ auth_action ──── auth_action_group_member (self, DAG)
                  ├─ auth_resource ─┬─ auth_resource_parent (self, DAG)
                  │                 ├─ auth_resource_attribute
                  │                 ├─ auth_page ──── auth_page_element
                  │                 └─ auth_api
                  ├─ auth_principal ┬─ auth_principal_parent (self, DAG)
                  │                 └─ auth_principal_attribute
                  ├─ auth_policy ───┬─ auth_policy_version
                  │                 └─ auth_policy_template_link
                  ├─ auth_policy_set_version ──── auth_policy_set_member
                  └─ auth_decision_log
```

DDL below is ANSI-flavored. For Oracle substitute `VARCHAR2`, `NUMBER(19)`, `CLOB`, and identity via sequence or `GENERATED BY DEFAULT AS IDENTITY`; for PostgreSQL use `BIGSERIAL` / `TEXT` / `JSONB`.

### 5.2 Registry and schema

```sql
CREATE TABLE auth_application (
  app_id          BIGINT       PRIMARY KEY,
  app_key         VARCHAR(64)  NOT NULL,          -- 'billing-portal'
  cedar_namespace VARCHAR(128) NOT NULL,          -- 'BillingPortal'
  description     VARCHAR(512),
  status          VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  created_at      TIMESTAMP    NOT NULL,
  created_by      VARCHAR(128) NOT NULL,
  updated_at      TIMESTAMP,
  updated_by      VARCHAR(128),
  CONSTRAINT uq_app_key UNIQUE (app_key)
);

CREATE TABLE auth_cedar_schema (
  schema_id    BIGINT      PRIMARY KEY,
  app_id       BIGINT      NOT NULL REFERENCES auth_application(app_id),
  version_no   INTEGER     NOT NULL,
  schema_json  CLOB        NOT NULL,   -- Cedar JSON schema, generated from the catalog
  status       VARCHAR(16) NOT NULL,   -- DRAFT | PUBLISHED | RETIRED
  content_hash CHAR(64)    NOT NULL,
  published_at TIMESTAMP,
  published_by VARCHAR(128),
  CONSTRAINT uq_schema_ver UNIQUE (app_id, version_no)
);
```

### 5.3 Actions

```sql
CREATE TABLE auth_action (
  action_id    BIGINT       PRIMARY KEY,
  app_id       BIGINT       NOT NULL REFERENCES auth_application(app_id),
  action_key   VARCHAR(128) NOT NULL,   -- 'approveInvoice'
  cedar_uid    VARCHAR(512) NOT NULL,   -- 'BillingPortal::Action::"approveInvoice"'
  action_kind  VARCHAR(16)  NOT NULL,   -- ATOMIC | GROUP
  risk_level   VARCHAR(16)  NOT NULL DEFAULT 'NORMAL',  -- NORMAL | ELEVATED | RESTRICTED
  display_name VARCHAR(256),
  description  VARCHAR(1024),
  status       VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  created_at   TIMESTAMP    NOT NULL,
  created_by   VARCHAR(128) NOT NULL,
  CONSTRAINT uq_action_key  UNIQUE (app_id, action_key),
  CONSTRAINT ck_action_kind CHECK (action_kind IN ('ATOMIC','GROUP'))
);

-- Action hierarchy: 'approveInvoice' in 'ApproverActions' in 'WriteActions'
CREATE TABLE auth_action_group_member (
  parent_action_id BIGINT NOT NULL REFERENCES auth_action(action_id),
  child_action_id  BIGINT NOT NULL REFERENCES auth_action(action_id),
  PRIMARY KEY (parent_action_id, child_action_id),
  CONSTRAINT ck_no_self_parent CHECK (parent_action_id <> child_action_id)
);
```

Cycle prevention in the action DAG is enforced by the service layer (topological check on insert), not by the database.

### 5.4 Resources (shared supertype)

Pages and APIs are both Cedar entities and both need a UID, attributes, and a place in the containment hierarchy. Rather than duplicate those columns, a shared `auth_resource` table carries them and the subtype tables carry the surface-specific columns.

```sql
CREATE TABLE auth_resource (
  resource_id   BIGINT       PRIMARY KEY,
  app_id        BIGINT       NOT NULL REFERENCES auth_application(app_id),
  resource_kind VARCHAR(32)  NOT NULL,   -- PAGE | PAGE_ELEMENT | API | MODULE | API_GROUP | DOMAIN
  cedar_type    VARCHAR(128) NOT NULL,   -- 'Page' | 'Feature' | 'Api' | 'Module' | 'ApiGroup'
  resource_key  VARCHAR(256) NOT NULL,   -- natural id used as the Cedar entity id
  cedar_uid     VARCHAR(512) NOT NULL,   -- 'BillingPortal::Page::"invoice.list"'
  display_name  VARCHAR(256),
  status        VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  created_at    TIMESTAMP    NOT NULL,
  created_by    VARCHAR(128) NOT NULL,
  CONSTRAINT uq_resource UNIQUE (app_id, cedar_type, resource_key)
);
CREATE INDEX ix_resource_uid ON auth_resource (cedar_uid);

-- Containment hierarchy backing Cedar's `resource in ...`
CREATE TABLE auth_resource_parent (
  child_resource_id  BIGINT NOT NULL REFERENCES auth_resource(resource_id),
  parent_resource_id BIGINT NOT NULL REFERENCES auth_resource(resource_id),
  PRIMARY KEY (child_resource_id, parent_resource_id),
  CONSTRAINT ck_res_no_self CHECK (child_resource_id <> parent_resource_id)
);
CREATE INDEX ix_res_parent_rev ON auth_resource_parent (parent_resource_id);

-- ABAC attributes projected into Cedar entity attributes at request time
CREATE TABLE auth_resource_attribute (
  resource_id BIGINT        NOT NULL REFERENCES auth_resource(resource_id),
  attr_name   VARCHAR(64)   NOT NULL,   -- 'classification', 'tenantId'
  attr_type   VARCHAR(16)   NOT NULL,   -- STRING | LONG | BOOL | SET_STRING | ENTITY
  attr_value  VARCHAR(2000) NOT NULL,
  PRIMARY KEY (resource_id, attr_name),
  CONSTRAINT ck_attr_type CHECK (attr_type IN ('STRING','LONG','BOOL','SET_STRING','ENTITY'))
);
```

### 5.5 Pages

```sql
CREATE TABLE auth_page (
  page_id             BIGINT       PRIMARY KEY,
  resource_id         BIGINT       NOT NULL REFERENCES auth_resource(resource_id),
  app_id              BIGINT       NOT NULL REFERENCES auth_application(app_id),
  page_key            VARCHAR(128) NOT NULL,   -- 'invoice.list'
  route_path          VARCHAR(512) NOT NULL,   -- '/billing/invoices'
  module_key          VARCHAR(128),            -- denormalized from auth_resource_parent
  menu_parent_page_id BIGINT       REFERENCES auth_page(page_id),
  menu_label          VARCHAR(256),
  menu_order          INTEGER,
  is_menu_item        CHAR(1)      NOT NULL DEFAULT 'Y',
  required_action_id  BIGINT       NOT NULL REFERENCES auth_action(action_id),
  status              VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  CONSTRAINT uq_page_resource UNIQUE (resource_id),
  CONSTRAINT uq_page_key      UNIQUE (app_id, page_key),
  CONSTRAINT uq_page_route    UNIQUE (app_id, route_path)
);

-- Sub-page gating: buttons, tabs, grid columns, widgets
CREATE TABLE auth_page_element (
  element_id         BIGINT       PRIMARY KEY,
  page_id            BIGINT       NOT NULL REFERENCES auth_page(page_id),
  resource_id        BIGINT       NOT NULL REFERENCES auth_resource(resource_id),
  element_key        VARCHAR(128) NOT NULL,   -- 'btn.approve'
  element_type       VARCHAR(32)  NOT NULL,   -- BUTTON | TAB | COLUMN | WIDGET | MENU
  required_action_id BIGINT       NOT NULL REFERENCES auth_action(action_id),
  status             VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  CONSTRAINT uq_element_resource UNIQUE (resource_id),
  CONSTRAINT uq_element_key      UNIQUE (page_id, element_key)
);
```

### 5.6 APIs

```sql
CREATE TABLE auth_api (
  api_id             BIGINT       PRIMARY KEY,
  resource_id        BIGINT       NOT NULL REFERENCES auth_resource(resource_id),
  app_id             BIGINT       NOT NULL REFERENCES auth_application(app_id),
  api_key            VARCHAR(128) NOT NULL,   -- 'invoice.approve.v1'
  service_key        VARCHAR(64),             -- owning microservice
  http_method        VARCHAR(10)  NOT NULL,   -- GET | POST | PUT | PATCH | DELETE
  path_template      VARCHAR(512) NOT NULL,   -- '/api/v1/invoices/{invoiceId}/approve'
  match_order        INTEGER      NOT NULL DEFAULT 100,  -- tie-break for overlapping templates
  required_action_id BIGINT       NOT NULL REFERENCES auth_action(action_id),
  -- Optional: enables the fine-grained object check
  domain_cedar_type  VARCHAR(128),            -- 'Invoice'
  domain_id_source   VARCHAR(128),            -- 'path:invoiceId' | 'body:$.invoiceId' | 'query:id'
  is_public          CHAR(1)      NOT NULL DEFAULT 'N',  -- unauthenticated allowlist
  status             VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  CONSTRAINT uq_api_resource UNIQUE (resource_id),
  CONSTRAINT uq_api_key      UNIQUE (app_id, api_key),
  CONSTRAINT uq_api_route    UNIQUE (app_id, http_method, path_template)
);
CREATE INDEX ix_api_lookup ON auth_api (app_id, http_method, status);

-- Which APIs a page calls. Not used at decision time; used by the consistency
-- linter that flags a page reachable by a role whose APIs that role cannot call.
CREATE TABLE auth_page_api (
  page_id BIGINT NOT NULL REFERENCES auth_page(page_id),
  api_id  BIGINT NOT NULL REFERENCES auth_api(api_id),
  PRIMARY KEY (page_id, api_id)
);
```

`match_order` matters because `/api/v1/invoices/summary` and `/api/v1/invoices/{invoiceId}` both match the same request. The route resolver loads all rows into an in-memory radix trie at startup, preferring literal segments over parameters, with `match_order` breaking any remaining ties.

**A request that matches no `auth_api` row is denied.** That single rule turns the catalog into a deployment gate for new endpoints rather than documentation that drifts.

### 5.7 Principals

```sql
CREATE TABLE auth_principal (
  principal_id   BIGINT       PRIMARY KEY,
  app_id         BIGINT       REFERENCES auth_application(app_id),  -- NULL = global
  principal_kind VARCHAR(32)  NOT NULL,   -- USER | SERVICE | GROUP | ROLE
  cedar_type     VARCHAR(128) NOT NULL,   -- 'User' | 'Group'
  principal_key  VARCHAR(256) NOT NULL,   -- IdP subject claim, or group name
  cedar_uid      VARCHAR(512) NOT NULL,
  source_system  VARCHAR(32)  NOT NULL,   -- IDP | LOCAL
  status         VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
  last_synced_at TIMESTAMP,
  CONSTRAINT uq_principal UNIQUE (app_id, cedar_type, principal_key)
);
CREATE INDEX ix_principal_uid ON auth_principal (cedar_uid);

CREATE TABLE auth_principal_parent (
  child_principal_id  BIGINT      NOT NULL REFERENCES auth_principal(principal_id),
  parent_principal_id BIGINT      NOT NULL REFERENCES auth_principal(principal_id),
  source_system       VARCHAR(32) NOT NULL,
  PRIMARY KEY (child_principal_id, parent_principal_id)
);
CREATE INDEX ix_principal_parent_rev ON auth_principal_parent (parent_principal_id);

CREATE TABLE auth_principal_attribute (
  principal_id BIGINT        NOT NULL REFERENCES auth_principal(principal_id),
  attr_name    VARCHAR(64)   NOT NULL,
  attr_type    VARCHAR(16)   NOT NULL,
  attr_value   VARCHAR(2000) NOT NULL,
  PRIMARY KEY (principal_id, attr_name)
);
```

### 5.8 Policy store

```sql
CREATE TABLE auth_policy (
  policy_id      BIGINT        PRIMARY KEY,
  app_id         BIGINT        NOT NULL REFERENCES auth_application(app_id),
  policy_key     VARCHAR(128)  NOT NULL,   -- stable Cedar policy id: 'F-901-mfa-for-restricted'
  policy_kind    VARCHAR(16)   NOT NULL,   -- STATIC | TEMPLATE
  effect         VARCHAR(8)    NOT NULL,   -- PERMIT | FORBID  (denormalized for search)
  policy_text    CLOB          NOT NULL,   -- Cedar source; the source of truth
  policy_json    CLOB,                     -- Cedar JSON form, cached for programmatic edit
  category       VARCHAR(32),              -- GUARDRAIL | ROLE | DELEGATION | EXCEPTION
  description    VARCHAR(2000) NOT NULL,
  owner_team     VARCHAR(128)  NOT NULL,
  status         VARCHAR(16)   NOT NULL,   -- DRAFT | ACTIVE | SUSPENDED | RETIRED
  effective_from TIMESTAMP,
  effective_to   TIMESTAMP,
  version_no     INTEGER       NOT NULL DEFAULT 1,
  created_at     TIMESTAMP     NOT NULL,
  created_by     VARCHAR(128)  NOT NULL,
  updated_at     TIMESTAMP,
  updated_by     VARCHAR(128),
  CONSTRAINT uq_policy_key  UNIQUE (app_id, policy_key),
  CONSTRAINT ck_policy_kind CHECK (policy_kind IN ('STATIC','TEMPLATE')),
  CONSTRAINT ck_effect      CHECK (effect IN ('PERMIT','FORBID'))
);

-- Full history; every UPDATE to auth_policy writes the prior row here.
CREATE TABLE auth_policy_version (
  policy_version_id BIGINT        PRIMARY KEY,
  policy_id         BIGINT        NOT NULL REFERENCES auth_policy(policy_id),
  version_no        INTEGER       NOT NULL,
  policy_text       CLOB          NOT NULL,
  status            VARCHAR(16)   NOT NULL,
  change_reason     VARCHAR(1000),
  change_ticket     VARCHAR(64),
  changed_at        TIMESTAMP     NOT NULL,
  changed_by        VARCHAR(128)  NOT NULL,
  CONSTRAINT uq_policy_version UNIQUE (policy_id, version_no)
);

-- One row per grant. This is the "who can do what" table.
CREATE TABLE auth_policy_template_link (
  link_id            BIGINT        PRIMARY KEY,
  app_id             BIGINT        NOT NULL REFERENCES auth_application(app_id),
  template_policy_id BIGINT        NOT NULL REFERENCES auth_policy(policy_id),
  linked_policy_key  VARCHAR(160)  NOT NULL,  -- generated Cedar policy id
  principal_uid      VARCHAR(512),            -- fills ?principal (NULL if unused by template)
  resource_uid       VARCHAR(512),            -- fills ?resource
  justification      VARCHAR(1000),
  request_ticket     VARCHAR(64),
  granted_by         VARCHAR(128)  NOT NULL,
  granted_at         TIMESTAMP     NOT NULL,
  expires_at         TIMESTAMP,               -- time-bound grants
  revoked_at         TIMESTAMP,
  revoked_by         VARCHAR(128),
  status             VARCHAR(16)   NOT NULL,  -- ACTIVE | EXPIRED | REVOKED
  CONSTRAINT uq_link_key UNIQUE (app_id, linked_policy_key)
);
CREATE INDEX ix_link_principal ON auth_policy_template_link (principal_uid, status);
CREATE INDEX ix_link_resource  ON auth_policy_template_link (resource_uid, status);
CREATE INDEX ix_link_expiry    ON auth_policy_template_link (expires_at);
```

The two indexes on `principal_uid` and `resource_uid` are what make access reviews cheap: *"everything Alice has been granted"* and *"everyone who can touch the UK ledger"* become single index scans. That is the concrete payoff for keeping grants in tables rather than in policy files.

### 5.9 Publishing and audit

```sql
CREATE TABLE auth_policy_set_version (
  policy_set_id BIGINT       PRIMARY KEY,
  app_id        BIGINT       NOT NULL REFERENCES auth_application(app_id),
  version_no    BIGINT       NOT NULL,
  schema_id     BIGINT       NOT NULL REFERENCES auth_cedar_schema(schema_id),
  content_hash  CHAR(64)     NOT NULL,   -- SHA-256 over ordered policy texts + links
  policy_count  INTEGER      NOT NULL,
  link_count    INTEGER      NOT NULL,
  status        VARCHAR(16)  NOT NULL,   -- PUBLISHED | ROLLED_BACK
  published_at  TIMESTAMP    NOT NULL,
  published_by  VARCHAR(128) NOT NULL,
  CONSTRAINT uq_policy_set UNIQUE (app_id, version_no)
);

CREATE TABLE auth_policy_set_member (
  policy_set_id  BIGINT  NOT NULL REFERENCES auth_policy_set_version(policy_set_id),
  policy_id      BIGINT  NOT NULL REFERENCES auth_policy(policy_id),
  policy_version INTEGER NOT NULL,
  PRIMARY KEY (policy_set_id, policy_id)
);

CREATE TABLE auth_decision_log (
  decision_id          BIGINT        PRIMARY KEY,
  correlation_id       VARCHAR(64)   NOT NULL,
  occurred_at          TIMESTAMP     NOT NULL,
  app_id               BIGINT        NOT NULL,
  enforcement_point    VARCHAR(16)   NOT NULL,   -- UI | GATEWAY | SERVICE
  principal_uid        VARCHAR(512)  NOT NULL,
  action_uid           VARCHAR(512)  NOT NULL,
  resource_uid         VARCHAR(512)  NOT NULL,
  decision             VARCHAR(8)    NOT NULL,   -- ALLOW | DENY
  determining_policies VARCHAR(2000),            -- from Cedar diagnostics
  evaluation_errors    VARCHAR(2000),            -- ids of policies skipped on error
  policy_set_version   BIGINT        NOT NULL,
  latency_micros       INTEGER,
  shadow_mode          CHAR(1)       NOT NULL DEFAULT 'N',
  legacy_decision      VARCHAR(8)                -- for Phase 1 diffing
);
-- Partition by RANGE (occurred_at), monthly; retain per records-management policy.
CREATE INDEX ix_decision_principal ON auth_decision_log (principal_uid, occurred_at);
CREATE INDEX ix_decision_deny      ON auth_decision_log (decision, occurred_at);
```

Only DENY decisions and ELEVATED/RESTRICTED-risk ALLOWs are written synchronously; routine ALLOWs go to an async batched writer, to keep the hot path off the database.

---

## 6. Runtime Architecture

```
            ┌────────────────────────────────────────────────┐
            │  Policy Admin UI / API   (CRUD + publish)       │
            └───────────────┬────────────────────────────────┘
                            │ writes
                    ┌───────▼────────┐
                    │  Authorization │   auth_policy, auth_page,
                    │      Store     │   auth_api, auth_action, links
                    └───────┬────────┘
                            │ poll / outbox notify on version change
   ┌────────────────────────┼─────────────────────────┐
   │                        │                         │
┌──▼───────────┐   ┌────────▼────────┐      ┌─────────▼────────┐
│ BFF / UI PEP │   │  Gateway PEP    │      │  Service PEP     │
│ menu + page  │   │  API entry gate │      │  object-level    │
└──┬───────────┘   └────────┬────────┘      └─────────┬────────┘
   │                        │                         │
   └──────────► Embedded Cedar PDP (cedar-java) ◄──────┘
                 · policy set (versioned, in memory)
                 · route trie
                 · entity cache (Caffeine, short TTL)
```

**PDP deployment:** embedded as a library in each service, not a remote service. This removes a network hop from every API call and eliminates a shared availability dependency. A thin remote PDP is exposed only for the UI/BFF batch endpoints and for non-JVM consumers.

### Request flow — API check

1. Gateway filter extracts the JWT; `sub` → `User::"<sub>"`, group claims → `Group::"..."` parents.
2. Route resolver maps `(method, path)` to an `auth_api` row via the in-memory trie. **No match → 403.**
3. Resource = `Api::"<api_key>"`; action = the row's `required_action_id` UID.
4. Context assembled: `{ mfa, srcIp, tenantId }` from claims and request metadata.
5. Entity slice assembled: principal + group ancestors + attributes; resource + `ApiGroup` ancestors + attributes. Served from cache keyed on `(uid, policy_set_version)`.
6. `engine.isAuthorized(request, policySet, entities)` → Allow / Deny.
7. Deny → 403, with the determining policy id written to the log (never to the response body).
8. If `domain_cedar_type` is set on the row, the service performs the second, object-level check after loading the entity.

### Request flow — page and menu rendering

The UI needs the *set* of permitted pages, not one decision. `GET /api/authz/nav` returns allowed `page_key`s and `element_key`s by iterating `isAuthorized` over the candidate catalog. For a catalog of a few hundred pages and elements this is a tight in-memory loop with no I/O after the entity slice is fetched once — we expect low single-digit milliseconds for the whole response, but this must be benchmarked against our real policy count (§8).

The result is cached per `(principal_uid, group_set_hash, policy_set_version)` and invalidated automatically when the policy set version increments.

### Java integration sketch

```java
// Loaded once per policy-set version, held in an AtomicReference and swapped atomically.
record PolicySetSnapshot(long version, PolicySet policies, Schema schema) {}

@Component
public class CedarPdp {

    private final AuthorizationEngine engine = new BasicAuthorizationEngine();
    private final AtomicReference<PolicySetSnapshot> snapshot = new AtomicReference<>();
    private final EntityResolver entityResolver;   // cache-backed; reads the auth_* tables

    public Decision check(AuthzQuery q) {
        PolicySetSnapshot snap = snapshot.get();
        Entities entities = entityResolver.slice(q.principalUid(), q.resourceUid());

        AuthorizationRequest req = new AuthorizationRequest(
            EntityUid.parse(q.principalUid()).get(),
            EntityUid.parse(q.actionUid()).get(),
            EntityUid.parse(q.resourceUid()).get(),
            Context.fromMap(q.context())
        );

        AuthorizationResponse resp = engine.isAuthorized(req, snap.policies(), entities);
        return Decision.from(resp, snap.version());   // carries the determining policies
    }
}
```

> **SDK caveats.** The CedarJava API surface is still evolving — release 4.3.0 introduced `com.cedarpolicy.model.Context` and `com.cedarpolicy.model.entity.Entities` to replace the older `Map<String,Value>` and `Set<Entity>` parameters, and further changes are signposted. Pin a version and wrap the SDK behind our own `CedarPdp` interface so an upgrade is a one-file change. Note also that CedarJava ships a native library via `CedarJavaFFI`; the published `-uber.jar` bundles it, and container images must match the target architecture.

---

## 7. Policy Authoring and Governance

| Stage | Control |
|---|---|
| **Author** | Policy Admin UI writes `auth_policy` rows in `DRAFT`. Cedar parser validates syntax on save. |
| **Validate** | Cedar's schema validator checks the policy against the published `auth_cedar_schema` — catches references to actions or attributes that do not exist. |
| **Test** | Policy changes ship with regression cases. CI replays a sampled set of production decision-log records and diffs the outcome. |
| **Lint** | Custom checks: no unguarded attribute dereference in `forbid`; every `PERMIT` has a description and an owner; no policy grants `WriteActions` on `RESTRICTED` resources without an MFA condition. |
| **Review** | `GUARDRAIL`-category policies require security-team approval; `DELEGATION` grants require the resource owner. |
| **Publish** | Snapshot to `auth_policy_set_version`; PDPs pick up the new version within the poll interval. |
| **Roll back** | Re-publish a prior `policy_set_id` — a single atomic operation. |

Cedar's logical encoding also enables formal analysis, for example verifying that a policy-set refactor does not change the permissions actually authorized. That is worth wiring into CI once the policy set stabilizes, but it is not a Phase 1 dependency.

---

## 8. Performance and Caching

| Concern | Approach |
|---|---|
| Policy set load | Loaded once per version into memory. Parse cost is paid at publish, not per request. |
| Entity slice fetch | Caffeine cache keyed `(uid, policy_set_version)`; TTL 60s for principals, 10m for catalog resources. Group ancestry precomputed as a transitive-closure table refreshed on IdP sync. |
| Route resolution | Radix trie built at startup from `auth_api`; rebuilt on catalog version change. |
| Nav endpoint | Per-user cached response keyed on group-set hash + policy set version. |
| Decision logging | ALLOWs batched async; DENYs and elevated-risk ALLOWs written synchronously. |
| DB load | Steady-state authorization performs **zero** queries on the hot path once caches are warm. |

SLOs to validate in Phase 1: p99 added latency under 2 ms at the gateway PEP; nav endpoint p99 under 50 ms cold and under 5 ms cached.

---

## 9. Enforcement Coverage and the Fail-Closed Property

- Any HTTP request not matching an `auth_api` row is rejected. New endpoints are therefore *unreachable* until registered, which makes the catalog a review gate rather than stale documentation.
- A CI step compares Spring's `RequestMappingHandlerMapping` (and the UI's route table) against `auth_api` / `auth_page`, failing the build on unregistered routes.
- UI gating is a usability feature, never a security control. Every page-level restriction has a corresponding API-level restriction, and `auth_page_api` lets the linter prove it.

---

## 10. Delivery Plan

| Phase | Scope | Exit criteria |
|---|---|---|
| **0 — Inventory** (2–3 wks) | Build the schema; auto-extract routes into `auth_page` / `auth_api`; derive the action vocabulary from existing role checks. | Catalog covers 100% of routes; reviewed by app owners. |
| **1 — Shadow** (3–4 wks) | Embed the PDP; evaluate every request alongside the legacy check; log both to `auth_decision_log`. Enforce nothing. | Zero unexplained diffs over 2 weeks of production traffic; latency SLOs met. |
| **2 — API enforcement** (2–3 wks) | Flip the gateway PEP to enforcing, per service, behind a flag. Legacy checks remain as a backstop. | All services enforcing; no rollbacks for 2 weeks. |
| **3 — UI enforcement** (2 wks) | Nav and page-element rendering driven by `/api/authz/nav`. | Menu contents match API permissions; hardcoded UI role checks removed. |
| **4 — Cleanup** (2 wks) | Delete legacy permission tables and in-code role checks; decommission old admin screens. | Single source of truth; runbook and access-review reports live. |
| **5 — Enrichment** (ongoing) | ABAC conditions, time-bound delegation, object-level checks on high-risk flows, policy analysis in CI. | — |

Phase 1 is the phase that de-risks everything else, and it is the one most often cut short. It should not be.

---

## 11. Risks and Open Questions

### Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Incomplete entity slice (missing group parent or attribute) causes a wrong DENY | Outage for a user cohort | Shadow-mode diffing; alert whenever `evaluation_errors` is non-empty |
| `forbid` guardrail silently skipped on evaluation error → fail-open | Security bypass | Mandatory `has` guards, CI lint, monitor the error-skip rate |
| Policy set version drift across PDP nodes | Inconsistent decisions | Version reported in health check and stamped on every decision record; alert on divergence |
| Cache staleness after a revocation | Revoked access persists up to TTL | Short principal TTL (60s); explicit invalidation broadcast on revoke; state the SLA to the security team |
| CedarJava native library (JNI/FFI) packaging across environments | Deployment friction | Use the `-uber.jar`; pin the version; verify in CI on the target base image |
| Policy set growth from per-user template links | Evaluation slowdown | Prefer group-based static policies; monitor link count; Cedar slices the policy set per request, but this needs benchmarking at our scale |
| Default-deny breaks an unregistered legacy flow | Incident at cutover | Catalog completeness check in CI; per-service enforcement flags; fast rollback |

### Open questions for review

1. **Managed vs self-hosted.** Amazon Verified Permissions hosts Cedar policies and exposes management and decision APIs. Do we want that, or does the embedded-library model better fit our latency and data-residency constraints? *Current lean: embedded, with the store abstraction kept clean enough to migrate.*
2. **List filtering.** Cedar answers "can Alice see invoice 4711", not "which invoices can Alice see". Do we need policy-derived query predicates in Phase 1, or is post-filtering acceptable at our page sizes?
3. **Multi-tenancy.** One policy set per tenant, or one shared set with `tenantId` conditions? Shared is simpler and `F-900` covers isolation; per-tenant becomes necessary if tenants author their own policies.
4. **Group mastering.** Which groups come from the IdP and which are local to `auth_principal`? Mixed sources need an explicit precedence rule.
5. **Domain resource attributes.** For fine-grained checks, do we load domain entity attributes into Cedar entities at request time, or pass them through `context`? The former is more expressive; the latter avoids an extra fetch.
6. **Break-glass.** What is the emergency access path when the policy store is unavailable or a policy is wrong at 3am?

---

## 12. Worked Example

**Registering an endpoint**

```sql
-- 1. Action
INSERT INTO auth_action (action_id, app_id, action_key, cedar_uid, action_kind, risk_level, ...)
VALUES (501, 1, 'approveInvoice',
        'BillingPortal::Action::"approveInvoice"', 'ATOMIC', 'ELEVATED', ...);

INSERT INTO auth_action_group_member (parent_action_id, child_action_id)
VALUES (400 /* ApproverActions */, 501);

-- 2. Resource row for the endpoint
INSERT INTO auth_resource (resource_id, app_id, resource_kind, cedar_type, resource_key, cedar_uid, ...)
VALUES (9001, 1, 'API', 'Api', 'invoice.approve.v1',
        'BillingPortal::Api::"invoice.approve.v1"', ...);

INSERT INTO auth_resource_parent (child_resource_id, parent_resource_id)
VALUES (9001, 8010 /* ApiGroup::"invoice-write" */);

INSERT INTO auth_resource_attribute VALUES (9001, 'classification', 'STRING', 'RESTRICTED');
INSERT INTO auth_resource_attribute VALUES (9001, 'tenantId',       'STRING', 'ACME');

-- 3. The API row
INSERT INTO auth_api (api_id, resource_id, app_id, api_key, service_key, http_method,
                      path_template, required_action_id, domain_cedar_type, domain_id_source, ...)
VALUES (701, 9001, 1, 'invoice.approve.v1', 'billing-svc', 'POST',
        '/api/v1/invoices/{invoiceId}/approve', 501, 'Invoice', 'path:invoiceId', ...);
```

**Granting Alice approver rights for the UK ledger only**

```sql
INSERT INTO auth_policy_template_link (
  link_id, app_id, template_policy_id, linked_policy_key,
  principal_uid, resource_uid,
  justification, request_ticket, granted_by, granted_at, expires_at, status)
VALUES (
  30001, 1, 2020 /* T-020-approver */, 'T-020-approver::alice::uk-ledger',
  'BillingPortal::User::"alice@example.com"',
  'BillingPortal::Module::"ledger-uk"',
  'Quarter-end approval coverage', 'ACC-8842', 'manager@example.com',
  CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + INTERVAL '90' DAY, 'ACTIVE');
```

At publish time this becomes the Cedar template-linked policy:

```cedar
// derived from T-020-approver
permit (
  principal == BillingPortal::User::"alice@example.com",
  action in BillingPortal::Action::"ApproverActions",
  resource in BillingPortal::Module::"ledger-uk"
);
```

**Resulting decision for `POST /api/v1/invoices/4711/approve`**

*Coarse check:* `(User::"alice@…", Action::"approveInvoice", Api::"invoice.approve.v1")` — permitted by the linked policy **if** `Api::"invoice.approve.v1"` sits in the subtree of `Module::"ledger-uk"`, and not forbidden by `F-901` provided `context.mfa == true`.

*Fine check in the service:* `(User::"alice@…", Action::"approveInvoice", Invoice::"4711")` — additionally subject to `F-900` tenant isolation and the `F-902` approval ceiling.

---

## 13. References

- Cedar documentation — https://docs.cedarpolicy.com/
- Terminology and concepts — https://docs.cedarpolicy.com/overview/terminology.html
- Authorization semantics (default-deny, forbid-wins, skip-on-error) — https://docs.cedarpolicy.com/auth/authorization.html
- Policy templates — https://docs.cedarpolicy.com/policies/templates.html
- Best practice: roles with policy templates — https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles-templates.html
- Best practice: representing relationships — https://docs.cedarpolicy.com/bestpractices/bp-relationship-representation.html
- JSON policy format — https://docs.cedarpolicy.com/policies/json-format.html
- Cedar implementation (Rust) — https://github.com/cedar-policy/cedar
- CedarJava bindings — https://github.com/cedar-policy/cedar-java
- Maven artifact — `com.cedarpolicy:cedar-java` (4.10.0)
- Cedar language paper, OOPSLA 2024 — https://dl.acm.org/doi/10.1145/3649835
- Amazon Verified Permissions (managed option) — https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/policy-templates.html
