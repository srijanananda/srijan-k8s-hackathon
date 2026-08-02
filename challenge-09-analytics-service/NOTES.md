# Challenge 9: Analytics Service - Ordered Data Processing

## What was done
- Created srijan-analytics-svc headless Service (clusterIP: None) for stable
  per-pod DNS identities, same pattern as Challenge 4's MySQL service
- Created srijan-analytics StatefulSet (3 replicas, hashicorp/http-echo image
  simulating an analytics processor)
- Applied storageClassName: gp2 explicitly in volumeClaimTemplates from the
  start (lesson learned from Challenge 4/6's PVC binding issues) - all 3 PVCs
  bound instantly with zero troubleshooting this time
- Connected to Challenge 4's database by reusing Challenge 5's existing
  srijan-db-secret (MYSQL_USER/MYSQL_PASSWORD) and srijan-app-config
  (DATABASE_URL) rather than duplicating credentials
- Verified ordered, sequential startup: srijan-analytics-0 reached Running
  before srijan-analytics-1 began creating, which completed before
  srijan-analytics-2 began - confirms StatefulSet's default ordered
  guarantee (contrast with Deployments, which start all replicas in parallel)
- Verified 3 independent PVCs (srijan-analytics-data-srijan-analytics-0/1/2),
  each a separate 2Gi Bound gp2 volume
- Verified stable DNS identity for all 3 replicas via getent hosts from a
  frontend pod - each resolved to a distinct, correct pod IP
