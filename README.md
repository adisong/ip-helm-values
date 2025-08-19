# ip-helm-values

This repo contains only helm values for applications. Each app would have it's own folder.

There is not a lot of values added here to simplify this setup and make sure it boots on my single node kind cluster but in production environment I would add following improvements:

1. Setup PDBs to limit number of disruptions during cluster events like node upgrade. I'm using `maxUnavailable` rather than `minAvailable` as it is generally safer. Both of those values are rounded up to the nearest integer so for low number of replicas `maxUnavailable` would always be at least 1 and won't cause cluster to effectively deadlock waiting for PDB that can't be fulfilled. GKE in particular would obey PDBs rigorously to the point it would wait for Pods to just die naturally which could cause nodePool upgrade to take hours if not days.

```yaml
pdb:
  enabled: true
  maxUnavailable: 25%
```

2. Setup `topologySpreadConstraints` to spread Pods to multiple AZs for HA.

```yaml
topologySpreadConstraints:
  - labelSelector:
      matchLabels:
        app.kubernetes.io/name: sampleapp
        app.kubernetes.io/instance: sampleapp
    maxSkew: 1
    whenUnsatisfiable: "DoNotSchedule"
    topologyKey: "topology.kubernetes.io/zone"
```

3. With Pods being spread across multiple AZs there would be an increase of inter-zone traffic which could potentially be reduced with `trafficDistribution` feature. This is relatively new feature and chart is configured to enable it only if k8s 1.33 is being used which is when this feature graduated to GA. It will prioritize traffic in the same AZ (which does not incure additional cost) and if no instances are available in the same AZ it would route to other AZ as normal.

```yaml
service:
  trafficDistribution:
    enabled: true
    strategy: "PreferClose"
```