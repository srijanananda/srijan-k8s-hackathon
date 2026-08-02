# Challenge 6: Resource Management - Performance Optimization

## What was done
- Added resource requests/limits to frontend Deployment (100m/200m CPU, 128Mi/256Mi memory)
- Added resource requests/limits to backend-api Deployment (200m/400m CPU, 256Mi/512Mi memory)
- Updated MySQL StatefulSet resources to match spec (500m/1000m CPU, 1Gi/2Gi memory)
- Created srijan-prod-quota ResourceQuota (4 CPU / 8Gi memory hard limits) on srijan-prod
- Tested enforcement: a pod requesting 5 CPU / 10Gi memory was correctly rejected
  at admission time with "Forbidden: exceeded quota"

## Issues encountered & resolved
1. First apply of the backend Deployment failed with a strict decoding error -
   the `resources` block was mistakenly nested under `volumes` instead of inside
   the container spec. Kubernetes rejected the whole apply rather than silently
   misapplying it (old pods kept running, unaffected). Fixed by relocating
   `resources` under the correct container block; reapplied successfully,
   confirmed via `describe deployment` (correct Limits/Requests shown) and a
   clean rollout to 3/3 new pods.
2. Discovered the StatefulSet's PVC bound to the manually-created static PV
   (srijan-static-pv, hostPath-backed) instead of dynamically provisioning a
   new EBS volume via the gp2 StorageClass - Kubernetes prefers matching an
   existing compatible Available PV over dynamic provisioning. Practical
   implication: MySQL data currently sits on node-local hostPath storage
   (less durable, tied to a specific node) rather than a real EBS volume.
   Addressed in a follow-up fix: deleted srijan-static-pv and recreated the
   MySQL pod to force genuine dynamic EBS provisioning going forward.
