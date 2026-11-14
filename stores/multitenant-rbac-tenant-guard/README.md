# Multi-Tenant RBAC with Tenant Guard

This sample store demonstrates how to enforce **tenant isolation as a model property** in an OpenFGA Role-Based Access Control (RBAC) architecture.

## Use-Case

In a standard multi-tenant RBAC system (such as [multitenant-rbac](../multitenant-rbac)), tenants are modeled as `organization` objects with associated roles, groups, and resources. However, in an unguarded model, OpenFGA evaluates tuples strictly based on graph connectivity: if a mistaken or malicious tuple assigns a role from Tenant A to an organization relation on Tenant B, OpenFGA's `Check` API will honor that relationship. Tenant boundaries in that scenario rely entirely on conventions within the calling application.

This store shows a **tenant-guard** pattern:
Every permission on an organization or tenant-owned resource requires, in addition to the role-derived relation, that the principal be a member of the resource's owning organization (`... and member`). With this guard in place, any accidental or malicious cross-tenant binding grants zero permissions to users outside that tenant.

## How the guard works

- `organization#member` is the tenant roster, and it accepts **direct users only** (`[user]`). Groups and roles never confer tenant membership; they only derive capabilities for principals who are already on the roster. Every capability on an organization (`admin`, `document_manager`, …) is defined as `(<role assignees> or <inherited>) and member`, so a principal must be on the roster *and* hold a role to exercise anything.
- Resources (`document`) derive permissions only `from organization`, so the guard applies to every tenant-owned resource without repeating it per resource type.
- Because the roster is direct-only, removing a user's single `member` tuple revokes all of their access in that organization at once, regardless of roles or groups — a useful property for offboarding. (If the roster also accepted `group#member`, a user could remain a member through a group and this would not hold; that is why the model does not allow it.)
- `role#organization` and `group#organization` record which tenant owns a role or group. They are **not** used in evaluation; they exist so the application can validate relationship writes (see below).

## What this model guarantees

- **Cross-tenant isolation against foreign role bindings**: If a role from Organization A is mistakenly bound to a capability on Organization B (e.g., `role:acme-document-management#assignee` bound to `organization:globex#document_manager`), users who belong only to Organization A are denied (`can_view: false`, `can_edit: false`).
- **Resource permission derivation through tenant chain**: Resources (`document`) derive their permissions strictly through their parent organization relation (`define organization: [organization]`).
- **Scoped admin privileges**: An organization admin in Tenant A holds no administrative or resource permissions in Tenant B.
- **Safe reverse queries**: Reverse queries (`list_objects` and `list_users`) respect the intersection guard:
  - `list_objects` for a user in Tenant A will never return resources belonging to Tenant B, even in the presence of invalid cross-tenant role bindings.
  - `list_users` on Tenant B's resources strictly returns Tenant B users and omits foreign principals.
- **Support for legitimate multi-tenant users**: Users who legitimately hold memberships in multiple organizations can access resources in each organization according to their assigned roles in each tenant, without permission leaking across boundaries.
- **Groups cannot enlarge the roster**: since `member` only accepts `user`, a mis-nested or cross-tenant group (such as the deliberate `group:acme-engineering#member → group:globex-engineering` tuple in the fixtures) can never add anyone to a tenant's membership; the type restriction makes attaching a group to `organization#member` unwritable.

## What it does not guarantee and why

While the tenant guard provides defense-in-depth at authorization check time, the OpenFGA Domain Specific Language (DSL) has specific structural boundaries that must be complemented by **write-time validation** when creating relationships:

1. **Role and group tenant affinity cardinality**:
   The model declares `define organization: [organization]` on `role` and `group` to express ownership, but the language cannot enforce cardinality ("exactly one owning organization", "never orphaned").
   * *Enforce when writing relationships*: Reject any role or group creation tuple that lacks a valid owning organization.

2. **Cross-tenant role binding when a user belongs to both tenants**:
   Because the DSL cannot express cross-object attribute unification (i.e., verifying that `role.organization == resource.organization` dynamically during traversal), a user who legitimately belongs to *both* Tenant A and Tenant B could inherit permissions if Tenant A's role is bound to Tenant B's organization.
   * *Enforce when writing relationships*: Validate that when binding `role:X#assignee` to `organization:Y`, `role:X` is owned by `organization:Y`.

3. **Group and role nesting cycles / cross-tenant nesting**:
   OpenFGA's engine tolerates cyclic definitions gracefully (via internal cycle detection), but nesting roles or groups across tenants (e.g., `group:acme-eng#member` inside `group:globex-eng`) violates multi-tenant boundaries.
   * *Enforce when writing relationships*: Enforce a Directed Acyclic Graph (DAG) and reject nesting across different organization scopes at write time.

## Trade-offs: what the guard costs and when to use it

The guard is an intersection (`… and member`) on each organization capability. Intersections are more expensive for OpenFGA to evaluate than unions, and the cost shows up differently per API. The figures below were measured on OpenFGA v1.22.1 with PostgreSQL 16, query caches off, using two stores with identical fixtures whose only difference is the five `and member` clauses. The metric is datastore reads per request, which carries over to other deployments better than absolute latency; latency is given only as a ratio. The method and the full tables are in the pull request that added this store (openfga/sample-stores#76).

**Check.** Because the guard sits on organization capabilities (not on each resource) and `member` is a direct `[user]` relation, a Check pays a constant number of extra datastore reads however deep groups are nested: +2 reads for a single-path permission such as `can_edit`, +5 for `can_view`, at every group depth from 1 to 25. The serial critical path — the group chain — is unchanged; the extra reads run in parallel branches, so they show up mainly as datastore load (see *Concurrency*). For principals who are *not* on the roster, the guard short-circuits and the check costs roughly 45–88% *fewer* reads.

**ListObjects.** With the weighted-graph ListObjects pipeline that is enabled by default in v1.22.1, the intersection is resolved at the organization node before documents are streamed, so the guard's cost does not grow with the number of documents. On a clean tenant with 10,000 documents and group depth 5, guarded `can_edit` took 21 reads versus 20 unguarded, with identical result sets. On the tenant that carries the deliberately invalid cross-tenant tuples, read counts were the same at 100, 1,000 and 10,000 documents (31 and 31 for `can_edit`; 34 guarded versus 35 unguarded for `can_view`); there the unguarded store also returns the leaked foreign documents, so it does more work — a consequence of the leak, not a speed-up from the guard. If you run an older server or disable that pipeline, measure before relying on this.

**ListUsers.** This is where the guard costs. Reads stay constant (+2 for `can_edit`, +5 for `can_view`), but each `member` read fetches the organization's roster to intersect in memory, so the work grows with roster size: guarded median latency was 1.0–1.4× unguarded at 100 roster users and 1.5–2.1× at 1,000. For very large rosters combined with ListUsers-heavy traffic, prefer write-time validation or cache ListUsers results.

**Concurrency.** Against PostgreSQL with the default pool (`--datastore-max-open-conns=30`), the pool saturated at about 10 concurrent checks on the test machine. For an all-positive mix the guarded store does 36 versus 31 reads per check and sustained 5–21% lower throughput once saturated. With 30% negative checks, reads per check are about equal (30.5 versus 30.0) and throughput is comparable. With 50% negative checks the guarded store does fewer reads per check (26.7 versus 29.5) and sustained 6–20% *higher* throughput, because off-roster checks short-circuit.

**When to use the guard**

- Multiple services or teams write to the store, or tuple hygiene cannot be guaranteed.
- Capabilities are defined at the organization level and resources derive from it.
- A single-tuple revocation switch per tenant is valuable (offboarding).
- A meaningful share of checks are for principals outside the tenant (API keys, stale sessions, offboarded users), where the guard reduces load.

**When to rely on write-time validation alone**

- ListUsers-heavy workloads over tenants with very large rosters.
- Deep resource hierarchies where capabilities and `and member` guards would have to sit on thousands of intermediate folders or projects rather than once on `organization` (measure `ListObjects` first, since the flat read count above relies on resolving the intersection at the organization node).
- Saturated, all-positive Check workloads where a 5–20% throughput difference matters more than defense-in-depth.
- A single, well-tested write path already enforces tenant ownership.

The guard is defense-in-depth, and it fails closed: user provisioning must write `organization#member` before (or in the same batch as) role or group grants, or checks will return `false`. Because the guard sits on `organization` capabilities rather than on `document`, intentional cross-tenant sharing on individual resources (for example, an explicit `external_viewer: [user]` relation on `document`) remains possible — just avoid adding unguarded `role#assignee` or `group#member` grants directly on resource relations, which would bypass the organization-level guard. Regardless of whether the guard is used, the properties listed under "What it does not guarantee and why" must be enforced when relationships are written.

## Try It Out

1. Ensure the [FGA CLI](https://github.com/openfga/cli#installation) is installed.
2. In this store's directory, run:

```bash
fga model test --tests store.fga.yaml
```

## Related Stores

- [multitenant-rbac](../multitenant-rbac): The baseline multi-tenant RBAC pattern showing dynamic role assignments and nested groups.
- [custom-roles](../custom-roles): Demonstrates custom role definitions scoped to organizations.
