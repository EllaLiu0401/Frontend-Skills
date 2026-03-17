# Terraform Shared Resource Safety

## Context

Learned from a production incident where running `terraform apply` locally recreated an M2M (Machine-to-Machine) OAuth client but dropped its associated client grant, breaking CI for all PRs.

---

## Core Concepts

### M2M Client vs Client Grant

In OAuth2 / identity providers (e.g. Auth0, Okta), machine-to-machine communication relies on two linked resources:

| Resource | What It Is | Analogy |
|---|---|---|
| **M2M Client** | A machine identity with `client_id` + `client_secret` | Employee badge (identity) |
| **Client Grant** | Authorization linking that client to an API with specific scopes | Door access permissions on the badge |

Both live on the identity provider (remote). Both are managed by Terraform.

**If the client exists but the grant is missing → 403 Forbidden.** The machine can authenticate (prove who it is) but cannot authorize (prove what it's allowed to do).

### Why Recreating a Client Is Dangerous

```
Before:
  ✅ Client (client_id: "old123")
  ✅ Grant  (bound to "old123", scopes: [create:orgs, read:users])

Terraform recreates the client (destroy + create):
  ❌ Old client ("old123") destroyed → old grant destroyed with it
  ✅ New client ("new456") created
  ❌ New grant NOT created (apply interrupted / dependency issue)

Result:
  ✅ Client exists → can request a token
  ❌ No grant → identity provider returns 403
  ❌ All services depending on this client break
```

---

## The Incident Pattern

### What Happened

1. Developer modified Terraform config for the local dev environment (added callback URLs)
2. Ran `terraform apply` locally
3. Terraform decided to recreate (destroy + create) the management M2M client
4. The client grant was dropped during recreation
5. The local dev environment shares an identity provider tenant with CI
6. CI's E2E tests call the same identity provider API → 403 on org creation → all PRs failed

### Root Cause

- **Shared tenant**: Local dev and CI use the same identity provider tenant
- **No destruction guard**: Nothing prevented Terraform from destroying critical resources
- **Silent failure**: `terraform apply` succeeded (no error), but the grant was missing

---

## Prevention Strategies

### 1. Code-Level: `lifecycle { prevent_destroy = true }`

Add destruction guards to critical resources in Terraform:

```hcl
resource "some_provider_client" "management" {
  name     = "my-app-management"
  app_type = "non_interactive"
  # ... config ...

  lifecycle { prevent_destroy = true }
}
```

**Effect**: If Terraform tries to destroy this resource, it errors out immediately instead of proceeding silently.

```
Error: Instance cannot be destroyed

  resource "some_provider_client" "management" has
  lifecycle.prevent_destroy set, but the plan calls
  for this resource to be destroyed.
```

**When to use**: On any resource where accidental destruction would break shared environments (M2M clients, client grants, API resource servers, database clusters, etc.).

**If you legitimately need to recreate**: Temporarily remove the lifecycle block, apply, then add it back.

### 2. Process-Level: Always Plan Before Apply

```bash
# Step 1: ALWAYS plan first (read-only, safe)
terraform plan

# Step 2: Read the output carefully
#   ✅  ~ (update in-place)         → Safe, just updating attributes
#   ⚠️  -/+ (destroy then create)  → Dangerous! Investigate why
#   ❌  - (destroy)                 → Dangerous! Don't apply

# Step 3: Only apply if plan looks safe
terraform apply
```

### 3. Awareness: Shared vs Isolated Environments

| Environment | Who It Affects | Risk Level |
|---|---|---|
| Truly local (Docker, local DB) | Only you | Low |
| Shared dev tenant | All developers + CI | **High** |
| Staging | QA + pre-prod testing | **High** |
| Production | End users | **Critical** |

Before running `terraform apply`, always know which category your target falls into.

---

## Terraform Operations Cheat Sheet

| Operation | Risk | When to Use |
|---|---|---|
| `terraform plan` | None (read-only) | Always, before any apply |
| `terraform apply` (update in-place) | Low | Normal config changes |
| `terraform apply` (destroy + create) | **High** | Only after careful review |
| `terraform destroy` | **Critical** | Almost never in shared environments |

---

## Key Takeaways

1. **`terraform plan` is free and safe** — always run it before apply
2. **`terraform apply` on shared resources affects everyone** — treat it like a production deployment
3. **Add `prevent_destroy` to critical resources** — turn silent destruction into loud errors
4. **M2M Client without a Grant = 403** — both must exist and be correctly linked
5. **Destroy + Create ≠ Update** — recreation can break dependent resources even if the primary resource comes back
6. **CI failures can come from infrastructure changes**, not just code changes — if all PRs suddenly fail, check if someone applied Terraform recently

---

## Mental Model

> **Plan is free. Apply is a deployment. Destroy is irreversible.**
>
> If you see red (destroy) in the plan output — stop and think before proceeding.

---

*This pattern applies to any Infrastructure-as-Code tool (Terraform, Pulumi, CloudFormation) managing shared identity providers, databases, or API gateways.*
