# Challenge 3: Service Discovery - Internal Communication

## What was done
- Created ClusterIP service (srijan-frontend-svc) exposing Challenge 2's frontend
  deployment on port 80
- Deployed srijan-backend-api Deployment (3 replicas, hashicorp/http-echo image)
  simulating a lightweight backend service
- Created ClusterIP service (srijan-backend-svc) exposing backend-api on port 8080
- Verified DNS-based service discovery by exec-ing into a frontend pod and curling
  the backend via its cluster DNS name:
  srijan-backend-svc.srijan-dev.svc.cluster.local:8080 -> returned expected response
- Verified load balancing by inspecting the service's Endpoints object, confirming
  all 3 backend pod IPs were correctly registered on port 8080:
  192.168.43.26:8080, 192.168.77.237:8080, 192.168.86.13:8080

## Resources created
- srijan-frontend-service.yaml   (ClusterIP, port 80)
- srijan-backend-deployment.yaml (3 replicas, http-echo)
- srijan-backend-service.yaml    (ClusterIP, port 8080)

## Notes
- kubectl get endpoints showed a deprecation warning (v1 Endpoints deprecated in
  v1.33+, replaced by discovery.k8s.io/v1 EndpointSlice). Functionally fine for now;
  worth switching inspection commands to `kubectl get endpointslices` in later
  challenges if we want to stay ahead of the deprecation.
- Cluster running on t3.small x2 nodes (Free Tier, $0 compute), 8 pods currently
  running across frontend + backend, well within the 22 total pod-slot capacity.
