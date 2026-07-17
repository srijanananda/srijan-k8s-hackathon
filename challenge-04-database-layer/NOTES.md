# Challenge 4: Database Layer - Persistent Storage

## What was done
- Created a headless Service (srijan-mysql-svc, clusterIP: None) for stable
  per-pod DNS identity
- Deployed srijan-mysql StatefulSet (1 replica) using mysql:8.0
- Used a volumeClaimTemplate (5Gi, ReadWriteOnce) for the StatefulSet's real
  persistent storage - EKS's default gp2 StorageClass dynamically provisioned
  a backing AWS EBS volume automatically
- Also created a standalone 10Gi PersistentVolume (srijan-static-pv, hostPath)
  to satisfy the literal "create a PersistentVolume" requirement as a reference/
  demo object - note this PV is NOT what backs MySQL; the StatefulSet's own
  volumeClaimTemplate + dynamic provisioning is the actual working storage layer
  (the correct production pattern on EKS)
- Initialized the database via a ConfigMap (srijan-mysql-init-schema) mounted to
  /docker-entrypoint-initdb.d, auto-creating the asset_catalog table + 1 sample row
  on first boot
- Verified stable DNS identity: srijan-mysql-0.srijan-mysql-svc.srijan-dev.svc.cluster.local
  resolves correctly from other pods (confirmed via getent hosts from a frontend pod,
  since the mysql:8.0 image doesn't ship a hostname binary)
- Verified data: SELECT * FROM asset_catalog returned the expected seeded row

## Known item deferred to Challenge 5 (intentional)
- MYSQL_ROOT_PASSWORD is currently a plaintext env var in the StatefulSet YAML.
  This is insecure for a real/public repo and is intentionally being fixed in
  Challenge 5, which introduces a proper Kubernetes Secret for DB credentials.

