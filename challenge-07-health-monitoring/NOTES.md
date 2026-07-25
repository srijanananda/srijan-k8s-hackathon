# Challenge 7: Health Monitoring - Application Reliability

## What was done
- Created srijan-nginx-health-conf ConfigMap with a custom nginx server block
  adding two real HTTP endpoints: /health and /ready (both return 200 with
  plain text), mounted into the frontend container
- Added livenessProbe (GET /health) and readinessProbe (GET /ready) to the
  frontend Deployment
- Added livenessProbe (GET /health) and readinessProbe (GET /ready) to the
  backend-api Deployment - no config changes needed since hashicorp/http-echo
  responds 200 on any path by default
- Added startupProbe and livenessProbe to the MySQL StatefulSet using
  `mysqladmin ping` (exec-based, since MySQL has no HTTP interface).
  startupProbe allows up to 5 minutes (30 attempts x 10s) before liveness
  checks begin, preventing false-positive restarts during normal MySQL boot

## Failure simulation & verification
- Initial attempt: tried to break a running pod by deleting its mounted nginx
  config file in-container. Failed with "Device or resource busy" - this is
  expected behavior, since the config was mounted via subPath (a bind-mount
  of a single file, which Linux won't let you unlink while active) and
  ConfigMap volumes are read-only by design. Noted as a real Kubernetes
  constraint rather than a bug.
- Better simulation used instead: temporarily changed the frontend's
  readinessProbe path to a nonexistent endpoint and applied the change,
  simulating a bad deployment.
- Result: new replicas came up Running but stuck at 0/1 Ready (container
  alive, health check correctly failing). The RollingUpdate strategy
  (maxUnavailable: 1) automatically paused the rollout, keeping old healthy
  pods alive rather than replacing them with broken ones - confirmed via
  `kubectl get pods` showing a mix of old (1/1) and new (0/1) pods
  simultaneously.
- Confirmed via `kubectl get endpoints` that only the healthy old pods
  remained in the Service's routable endpoint list during the failure window
  - broken pods were correctly excluded from traffic despite being "Running"
- Reverted the probe path to the correct /ready endpoint; rollout completed
  cleanly, all 5 pods returned to 1/1 Running, and all 5 IPs returned to the
  Service's endpoint list

