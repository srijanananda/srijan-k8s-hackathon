# Challenge 8: External Access - Customer-Facing Gateway

## What was done
- Installed NGINX Ingress Controller (official v1.11.3 AWS provider manifest)
- Patched the controller's Service from the default LoadBalancer to NodePort
  to avoid provisioning a billed AWS Network Load Balancer (cost-conscious
  choice - NodePort uses existing node ports at $0 extra cost, at the expense
  of not having a stable public DNS/IP the way LoadBalancer provides)
- Generated a self-signed TLS certificate for host "srijan.local" using OpenSSL,
  stored as a kubernetes.io/tls Secret (srijan-tls-secret)
- Created srijan-ingress with two path-based rules under host srijan.local:
  /vault -> srijan-frontend-svc:80, /api -> srijan-backend-svc:8080
- Verified SSL termination, path-based routing, and HTTP->HTTPS redirect

## Exact steps for NodePort assignment, exposure, and verification

1. Install the controller:
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/aws/deploy.yaml

2. Confirm the controller pod is Running:
   kubectl get pods -n ingress-nginx
   (expect ingress-nginx-controller-* -> 1/1 Running)

3. Patch the Service type from LoadBalancer to NodePort:
   kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{\"spec\": {\"type\": \"NodePort\"}}'

4. Retrieve the assigned NodePorts:
   kubectl get svc ingress-nginx-controller -n ingress-nginx
   (PORT(S) column shows e.g. 80:32736/TCP,443:30830/TCP - record both numbers)

5. Create the TLS secret and apply the Ingress resource (see manifests below)

6. Verify the Ingress picked up the rules correctly:
   kubectl get ingress -n srijan-dev
   kubectl describe ingress srijan-ingress -n srijan-dev
   (confirm both /vault and /api show correct backend services + pod IPs,
   and the TLS section references srijan-tls-secret)

7. Verify routing + SSL end-to-end (see "External access testing" below)

## Debugging journey (in order encountered)

### Issue 1: External curl timed out on the NodePort
Symptom: curl: (28) Failed to connect... after ~21s timeout.
Diagnosis: AWS Security Groups act as an instance-level firewall. EKS's
auto-created security group only allows traffic from within the cluster/VPC
by default - checked via:
  aws ec2 describe-security-groups --region ap-south-1 --group-ids <sg-id> --query "SecurityGroups[].IpPermissions" --output json
Confirmed no rule existed allowing inbound traffic from any external IP on
the NodePort numbers.
Fix: added explicit inbound rules scoped to my own IP (not 0.0.0.0/0, for
security) on both NodePort numbers:
  aws ec2 authorize-security-group-ingress --region ap-south-1 --group-id <sg-id> --protocol tcp --port <port> --cidr "<my-ip>/32"

### Issue 2: Still timing out after the security group fix
Diagnosis: Home/mobile ISP IP addresses are dynamic and can change between
sessions. Re-checked current public IP via https://api.ipify.org and found
it had changed since the first fix.
Fix: re-ran the authorize-security-group-ingress commands with the current IP.

### Issue 3: Still timing out even with security group fully confirmed correct
Diagnosis process: verified internal cluster routing worked correctly first,
using a temporary debug pod to curl the Ingress controller from INSIDE the
cluster:
  kubectl run srijan-curl-test --image=curlimages/curl -n srijan-dev --rm -it --restart=Never -- curl -k https://ingress-nginx-controller.ingress-nginx.svc.cluster.local/vault -H "Host: srijan.local"
This succeeded (returned nginx HTML), proving the Ingress/Service/pod chain
was fully correct. Combined with the security group being independently
re-verified correct, this ruled out both the cluster config AND AWS-side
networking - pointing to the local network/ISP blocking outbound traffic to
non-standard high ports (30000-32767 range), a common restriction on home
routers/ISPs since legitimate consumer traffic rarely uses those ports.
Fix (pivot, not a "true fix" of the network restriction): used
`kubectl port-forward` to tunnel through the already-working Kubernetes API
server connection directly to the Ingress Service, avoiding the need for any
direct external connection to a node's public IP on a high port entirely:
  kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8443:443
This fully demonstrates the same Ingress/SSL/routing functionality without
depending on the local network's port restrictions.

### Issue 4: Response showed "Server: Apache" instead of nginx
Diagnosis: A pre-existing local development web server (Apache, likely via
XAMPP/WAMP/IIS) was already bound to port 8080 on the Windows machine,
colliding with a `kubectl port-forward ... 8080:80` attempt - Windows
allowed both to bind by using different interfaces (IPv6 vs IPv4), and curl
happened to hit the local Apache instance instead of the tunnel.
Fix: used an unused local port (18080) instead, confirming the correct
nginx-ingress response (308 redirect) rather than Apache's 404.

### Issue 5: Verbose curl output didn't show certificate subject/issuer lines
Diagnosis: Windows curl.exe uses the native `schannel` TLS library by default
rather than OpenSSL, which formats verbose (-v) output differently and does
not print subject:/issuer: lines the way OpenSSL-based curl builds do. This
is a tooling difference, not a sign SSL wasn't working.
Verification without those specific lines: confirmed successful TLS
handshake completion, explicit certificate renegotiation log lines
(`schannel: SSL/TLS connection renegotiated`), a valid 200 OK response
delivered over the encrypted channel, and presence of a
Strict-Transport-Security header (added by nginx-ingress specifically when
terminating SSL) - sufficient combined proof of working SSL termination.

## External access testing (final, working commands)

Terminal 1 (keep running):
  kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8443:443

Terminal 2:
  curl.exe -k --resolve srijan.local:8443:127.0.0.1 https://srijan.local:8443/vault
  curl.exe -k --resolve srijan.local:8443:127.0.0.1 https://srijan.local:8443/api

Result: /vault returned the nginx welcome page (frontend), /api returned
"Hello from srijan-backend-api" (backend) - both over HTTPS through the same
Ingress resource, confirming path-based routing + SSL termination together.

HTTP->HTTPS redirect test (separate port-forward to avoid the port 8080
conflict from Issue 4):
  kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 18080:80
  curl.exe -I --resolve srijan.local:18080:127.0.0.1 http://srijan.local:18080/vault
Result: HTTP/1.1 308 Permanent Redirect, Location: https://srijan.local/vault
confirming ssl-redirect annotation works correctly.

## NodePort vs LoadBalancer - why NodePort was chosen
- LoadBalancer: Kubernetes provisions a real AWS Network/Classic Load Balancer,
  billed continuously (~$0.006/hr + data processing) as long as it exists.
  Gives a stable public DNS name and is the standard production choice.
- NodePort: opens a port (30000-32767 range) directly on every node's existing
  IP, no new AWS resource created, $0 additional cost. Downside: no stable
  public endpoint, requires knowing a node's IP directly, less production-realistic.
- Chosen NodePort to stay consistent with this project's cost-conscious
  approach (Free Tier nodes, single NAT gateway) established from Challenge 1
  onward. In a real production deployment, this would use type: LoadBalancer
  or the AWS Load Balancer Controller with an ALB instead.

## Naming convention
CloudVault (original PDF) -> renamed to Srijan throughout ("srijan.local"
instead of "cloudvault.local").