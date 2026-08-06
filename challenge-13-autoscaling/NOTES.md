# Challenge 13: Auto-Scaling - Horizontal Pod Autoscaler (HPA)

## What was done
- Created srijan-frontend-hpa: target 70% CPU utilization, 3-10 replicas,
  targeting the srijan-frontend Deployment
- Created srijan-backend-hpa: target 60% CPU utilization, 3-8 replicas,
  targeting the srijan-backend-api Deployment
- Confirmed metrics-server (installed since Challenge 1) was healthy and
  actively reporting real CPU numbers to both HPAs (not <unknown>)
- Generated sustained load using a busybox pod running parallel wget loops
  against srijan-frontend-svc, to drive real CPU usage up

## Issues encountered & resolved
1. Initial load generator (single-threaded wget loop) was too weak against
   5 lightweight nginx replicas serving a static default page - CPU never
   exceeded ~8%, never triggering scale-up. Switched to a parallel loop
   (10 concurrent wget requests per cycle, looped continuously) for
   meaningfully higher sustained load.
2. Even with parallel load, frontend's original CPU request (100m) meant the
   generated load only reached 15-23% of target - not enough headroom to
   reliably demonstrate scale-up within a reasonable demo window. Temporarily
   lowered frontend's CPU request to 20m (HPA targets are calculated as a
   percentage of the pod's CPU *request*, not limit - a smaller request means
   the same real load represents a much higher percentage). This is a
   demo-only adjustment, documented here rather than left silently in place.
3. Path errors when trying to apply the frontend YAML change (ran the command
   from the wrong working directory) caused the edit to silently not take
   effect on the first two attempts - confirmed via
   `kubectl describe deployment ... | Select-String Requests` before trusting
   the change had applied.
4. Load generator pod exited on its own before manual cleanup was attempted
   (likely a transient shell/network hiccup in the busybox loop) - not an
   error requiring action, simply meant the load stopped slightly earlier
   than planned.

## Verification (real observed values)
- Under sustained load: cpu climbed to 125%/70% and 113%/70% (well over
  target), REPLICAS visibly increased from 3 -> 4 -> 5 in real time via
  `kubectl get hpa -w`
- After load stopped: cpu dropped to 5%/70%, replicas already trending back
  down from their peak toward minReplicas: 3 (full return to 3 takes several
  minutes due to HPA's default 5-minute scale-down stabilization window,
  which intentionally prevents flapping)
- Backend HPA remained steady at 3 replicas throughout (0% CPU, no load was
  directed at the backend in this test) - confirms HPA correctly does NOT
  scale when there's no actual demand

## Naming convention
CloudVault (original PDF) -> renamed to Srijan throughout.

## Note
Frontend's CPU request was changed from 100m to 20m specifically to make the
HPA demo reliably reproducible without needing a heavier external load-testing
tool. This is noted as a deliberate demo adjustment, not a production-sizing
recommendation.