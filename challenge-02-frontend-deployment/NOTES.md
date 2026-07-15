# Challenge 2: Frontend Deployment - Scalable Web Layer

## What was done
- Converted Challenge 1's standalone pod into a Deployment (srijan-frontend) with 3 replicas
- Configured RollingUpdate strategy (maxSurge: 1, maxUnavailable: 1)
- Performed rolling update: nginx:1.21 -> nginx:1.22 (zero downtime, verified via rollout status)
- Scaled to 5 replicas to simulate peak traffic

## Infra issue encountered & resolved
- t3.micro nodes hit AWS's "max pods per node" ENI-based limit (4 pods/node), causing
  2 of 3 replicas to stay Pending ("Too many pods").
- Migrated nodegroup to t3.small (still AWS Free Tier eligible, $0 compute cost),
  raising capacity to 11 pods/node. All replicas scheduled successfully after migration.
- Also hit a transient local DNS resolution timeout while creating the new nodegroup;
  resolved via flushdns + retry.

