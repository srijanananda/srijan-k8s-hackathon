# Challenge 11: Multi-Tenant Security - RBAC + Service Accounts

## What is RBAC and its components

RBAC (Role-Based Access Control) is Kubernetes's native permission system. It answers:
"Who (subject) can do what (verbs) to which resources in which namespace?"

**4 core objects:**

1. **ServiceAccount** — a pod-level identity (like a user account). Every pod has one
   (defaults to "default" if not specified). Used for:
   - RBAC: limiting what a pod can do via the Kubernetes API (main use in this challenge)
   - Pod identification: labeling/tracking which pod did what in logs
   - OIDC/IRSA: we used this earlier in Challenge 8 setup — the EBS CSI driver pod
     used a ServiceAccount linked to an AWS IAM Role, letting it call AWS APIs
   - Image pull secrets: ServiceAccounts can reference secrets for private registries
   - In short: ServiceAccount = "who is this pod, for security/tracking purposes"

2. **Role** — a set of permissions, scoped to ONE namespace. Defines the rules:
   "verbs (get/list/create/delete) on which resources (pods/secrets/deployments)"
   Example: "can get and list pods, but not delete them" — that's a Role

3. **RoleBinding** — glues a ServiceAccount to a Role within a namespace
   "ServiceAccount X gets the permissions defined in Role Y, in namespace Z"
   Without a RoleBinding, a ServiceAccount has zero permissions by default

4. **ClusterRole / ClusterRoleBinding** — same as Role/RoleBinding, but cluster-wide
   (all namespaces at once). We didn't use these because least-privilege means
   scoping permissions to only the namespaces they need

**How they work together:**

ServiceAccount (srijan-developer)
↓ (bound via RoleBinding)
Role (srijan-developer-role: can get/list pods only)
↓ (grants)
Permission to perform "get, list" on "pods" in namespace "srijan-dev"

## What we did


**Created 3 ServiceAccounts:**
- srijan-developer (namespace: srijan-dev) — least privileges
- srijan-operator (namespace: srijan-dev) — intermediate privileges
- srijan-admin (namespace: srijan-prod) — full access but scoped to prod only

**Created 3 Roles with different permission levels:**
- srijan-developer-role: get, list, watch on pods/deployments only (read-only, safe)
- srijan-operator-role: get, list, watch, create, update, patch, delete on
  pods/deployments/services/cronjobs/jobs (can manage the platform itself)
- srijan-admin-role: "*" (all verbs) on "*" (all resources) — full admin, but only
  in srijan-prod namespace (least-privilege even for admins — not cluster-wide)

**Bound them with RoleBindings:**
- developer-binding: links srijan-developer to srijan-developer-role in srijan-dev
- operator-binding: links srijan-operator to srijan-operator-role in srijan-dev
- admin-binding: links srijan-admin to srijan-admin-role in srijan-prod

**Verified with `kubectl auth can-i --as`:**
Tests show exactly which actions are allowed/denied for each ServiceAccount:

| ServiceAccount | Action | Namespace | Result | Proof |
|---|---|---|---|---|
| developer | get pods | srijan-dev | ✓ yes | read-only access works |
| developer | delete pods | srijan-dev | ✗ no | write-protected correctly |
| developer | get secrets | srijan-dev | ✗ no | secret access denied (security) |
| operator | delete cronjobs | srijan-dev | ✓ yes | can manage CronJobs from Ch10 |
| operator | get pods | srijan-prod | ✗ no | namespace boundary enforced |
| admin | delete deployments | srijan-prod | ✓ yes | full access in own namespace |
| admin | get pods | srijan-dev | ✗ no | scoped to prod only, not cluster-wide |

## Real-world analogy

Think of your company:
- **Developer ServiceAccount** = junior engineer, can view the dashboard but can't
  push to production (read-only safe)
- **Operator ServiceAccount** = SRE/platform engineer, can restart services and
  manage infrastructure (read+write, scoped)
- **Admin ServiceAccount** = team lead, full control but only over the production
  environment they're responsible for, not dev/staging

Each person (ServiceAccount) has a job title (Role) that defines what they can do.
The job assignment (RoleBinding) is separate from the job description (Role), so you
can easily reassign people to different roles without rewriting permissions.

