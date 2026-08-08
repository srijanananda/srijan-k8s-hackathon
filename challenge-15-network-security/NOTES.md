# Challenge 15: Network Security - NetworkPolicies + Final Validation

## What was done
- Verified VPC CNI network policy agent (aws-eks-nodeagent) was present, but
  discovered via `describe-addon` that enforcement was not actually enabled
  (configurationValues: None) - agent presence alone does not mean active
  enforcement
- Enabled enforcement: `aws eks update-addon --addon-name vpc-cni
  --configuration-values '{"enableNetworkPolicy":"true"}'`
- Created srijan-default-deny-ingress: podSelector {} with policyTypes
  [Ingress] and no ingress rules - blocks ALL inbound traffic to every pod
  in srijan-dev by default
- Created srijan-allow-ingress-to-frontend: permits traffic from the
  ingress-nginx namespace to frontend pods on port 80
- Created srijan-allow-frontend-to-backend: permits traffic from
  app=frontend pods to backend-api pods on port 8080 only
- Created srijan-allow-to-mysql: permits traffic from app=backend-api AND
  app=srijan-analytics pods to the MySQL pod on port 3306 only - no other
  source is permitted

## Verification (before vs after enforcement)
Before enabling enforcement: all test connections succeeded regardless of
policy, including ones that should have been blocked - confirming the
policies existed as objects but were not being acted on.

After enabling enforcement, re-ran identical tests using labeled busybox
pods (matching the real workload's labels, since NetworkPolicy matches by
label, not image):
| Test | Path | Result |
|---|---|---|
| frontend -> backend | permitted | SUCCESS (Hello from srijan-backend-api) |
| frontend -> MySQL | NOT permitted | BLOCKED (download timed out) |
| unlabeled pod -> MySQL | NOT permitted | BLOCKED (download timed out) |
| backend -> MySQL | permitted | SUCCESS (TCP connected, got MySQL protocol banner) |

The contrast between "connects immediately" and "hangs for the full 5s
timeout then fails" is the concrete evidence that default-deny plus
targeted allow rules are genuinely enforced, not just present as YAML.

## Key lesson
The presence of a CNI's network-policy-capable agent (aws-eks-nodeagent)
does not mean enforcement is active - this must be explicitly enabled via
addon configuration (enableNetworkPolicy: true for VPC CNI). Always verify
enforcement empirically (test both an allowed and a blocked path) rather
than assuming policies work just because `kubectl get networkpolicies`
shows them as created.

## Naming convention
CloudVault (original PDF) -> renamed to Srijan throughout.