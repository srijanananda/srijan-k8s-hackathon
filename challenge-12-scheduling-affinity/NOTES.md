# Challenge 12: Scheduling & Affinity - Node Selection & Pod Placement

## What was done
- Labeled all 3 nodes tier=general, then overwrote one specific node to tier=analytics
- Tainted that same node: workload=analytics:NoSchedule (repels any pod without
  a matching toleration)
- Added podAntiAffinity (preferredDuringScheduling) to frontend Deployment -
  prefers spreading replicas across different nodes, not required (since 5
  replicas across 2 available "general" nodes can't be perfectly 1-per-node)
- Added nodeAffinity (requiredDuringScheduling: tier=analytics) + a matching
  toleration for the workload=analytics taint to the analytics StatefulSet -
  hard requirement, must run on the labeled/tainted node only

## Issues encountered & resolved
1. PowerShell `foreach` loop over `kubectl get nodes -o jsonpath` output failed
   because the raw output is one space-separated string, not an array - the
   loop tried to label one node with all 3 names concatenated. Fixed by
   explicitly splitting the string (-split '\s+') before looping.
2. After applying node affinity, analytics-2 got stuck Pending with
   "PersistentVolume's node affinity" mismatch - its EBS volume (created
   before this challenge, in Challenge 9) was provisioned in a different
   availability zone than the node now required by the new affinity rule.
   EBS volumes are AZ-locked and cannot attach across zones. Diagnosed via
   `kubectl describe pod` Events section.
   Fixed by deleting the old PVCs and force-deleting the pods, letting the
   StatefulSet provision fresh volumes correctly matching the required node's
   AZ from the start.
3. One old PVC got stuck in Terminating state indefinitely due to the
   kubernetes.io/pvc-protection finalizer waiting for volume detachment
   confirmation. Verified via `kubectl get volumeattachments` that the old
   volume was still marked attached. Since the PVC was confirmed orphaned
   (its pod already had a new replacement volume), manually cleared the
   finalizer to force removal - only done because the PVC was verified unused,
   not a general-purpose fix.
4. A pod (analytics-1) continued running with a mount reference to a PVC
   object that no longer existed after the above cleanup - a "ghost" mount
   that Linux tolerates until the pod restarts. Force-deleted that pod to let
   the StatefulSet recreate it with a properly tracked, freshly provisioned PVC.

## Verification
- All 3 analytics pods confirmed running on the same node
  (ip-192-168-27-161, the tier=analytics/tainted node)
- All 3 analytics PVCs confirmed Bound with no orphaned/stuck entries
- All 5 frontend pods confirmed spread across the other 2 nodes only, with
  zero frontend pods scheduled onto the tainted analytics node - proving the
  taint correctly repelled workloads without a matching toleration

## Key lesson
Scheduling rules (affinity/taints) can conflict with resources that already
existed before the rule was introduced (e.g. an EBS volume tied to a specific
AZ). Any time a new placement rule is added, existing PVCs/PVs tied to
affected pods should be checked for compatibility, not assumed to adjust
automatically.

## Naming convention
CloudVault (original PDF) -> renamed to Srijan throughout.