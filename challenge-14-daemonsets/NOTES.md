# Challenge 14: Cluster-Wide Services - DaemonSets

## What was done
- Created srijan-fluentd DaemonSet (log collection) - mounts node's /var/log
  via hostPath, one pod per node automatically
- Created srijan-node-exporter DaemonSet (metrics) - uses hostNetwork: true
  and mounts node's /proc and /sys (read-only) to expose real node-level
  metrics on port 9100
- Both DaemonSets include an explicit toleration for the workload=analytics
  taint (from Challenge 12), since log/metrics collection needs visibility
  into every node including specially-tainted ones - this is the correct,
  intentional behavior for cluster-wide observability tooling

## Issues encountered & resolved
1. node-exporter pods stuck Pending (2 of 3) while fluentd scheduled cleanly
   on all 3 nodes immediately. Diagnosed via `kubectl describe pod` - found
   two distinct blockers layered together:
   a) "Too many pods" on one node - confirmed via `kubectl describe node`
      that all 3 nodes are hard-capped at 11 pods each (t3.small's real
      limit, consistent with original Challenge 1 assumption). One node had
      exactly 11/11 non-terminated pods running, mostly from HA-replicated
      system addons (coredns x2, ebs-csi-controller x2, metrics-server x2 -
      6 pods just from redundancy pairs unnecessary for a single-purpose
      demo cluster).
      Fixed by scaling ebs-csi-controller and metrics-server down from 2
      replicas to 1 each - safe since these are control-plane components,
      not the per-node ebs-csi-node/kubelet components, and losing HA
      redundancy is acceptable for a learning/demo cluster. Verified
      metrics-server still functioned correctly afterward via `kubectl get
      hpa` showing real (not <unknown>) values.
   b) Separately, the analytics node (already carrying MySQL, all 3 analytics
      replicas, and fluentd) was at 96% memory requests allocated
      (1384Mi/1440Mi), leaving insufficient headroom for node-exporter's
      original 64Mi memory request. Confirmed via `kubectl describe node`
      Allocated resources section. Fixed by lowering node-exporter's memory
      request from 64Mi to 32Mi (a legitimate right-sizing, since
      node-exporter is a lightweight process that reads /proc and /sys files
      and doesn't need significant memory in practice).
   Note: scheduler error messages mentioning "NodeAffinity" on the other 2
   nodes for each pending pod were a red herring, not a real problem -
   DaemonSets internally assign each pod an implicit per-node affinity as
   part of their "exactly one pod per node" guarantee, so seeing
   "NodeAffinity failed" on nodes other than a pod's designated target is
   expected and not the actual blocking reason.
2. Also temporarily scaled frontend back down from 5 to 3 replicas (matching
   HPA's own minReplicas from Challenge 13) to free general pod-count
   headroom across the cluster, since 5 was only needed for that challenge's
   scale-up demo and isn't required as a steady-state baseline.

## Verification
- srijan-fluentd: 3/3 Running, one per node, confirmed self-healing (deleted
  one pod manually, DaemonSet recreated it on the SAME node - unlike a
  Deployment, which could reschedule anywhere, a DaemonSet pod is tied to
  its specific node)
- srijan-node-exporter: 3/3 Running, one per node, including the
  resource-constrained analytics node
- Both DaemonSets automatically matched DESIRED=CURRENT=READY=3, tracking
  the cluster's node count without any replicas field being set anywhere -
  core DaemonSet behavior confirmed
