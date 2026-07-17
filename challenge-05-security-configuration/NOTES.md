# Challenge 5: Security Configuration - Secrets Management

## What was done
- Created srijan-db-secret (Opaque) holding mysql-root-password, mysql-user,
  mysql-password
- Created srijan-app-config ConfigMap holding database-url, cache-config,
  api-settings
- Updated Challenge 4's MySQL StatefulSet to consume MYSQL_ROOT_PASSWORD,
  MYSQL_USER, MYSQL_PASSWORD from the Secret instead of plaintext env values
- Updated Challenge 3's backend-api Deployment to:
  - consume DB credentials from the Secret as env vars
  - mount srijan-app-config as a volume at /etc/srijan-config
- Verified both workloads picked up the new config correctly after a full
  pod recreation cycle

## Issue encountered & resolved
- Applied the updated StatefulSet/Deployment YAML (referencing the new Secret/
  ConfigMap) BEFORE actually running `kubectl apply` on the Secret/ConfigMap
  YAML files themselves - out-of-order apply.
- Result: new backend-api pod stuck in ContainerCreating (FailedMount: configmap
  "srijan-app-config" not found), and MySQL pod logged repeated "secret
  srijan-db-secret not found" warnings during its restart window.
- Fix: applied srijan-db-secret.yaml and srijan-app-configmap.yaml, after which
  kubelet's automatic retry succeeded and both pods became Ready without any
  further manual intervention.
- Lesson: dependency order matters - Secrets/ConfigMaps must exist before
  workloads that reference them are applied/recreated.

