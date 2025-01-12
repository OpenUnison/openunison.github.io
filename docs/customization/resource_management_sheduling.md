# Resource Management and Scheduling

OpenUnison will consume 1-2 GB of memory.  CPU is much harder to track and should be set based on monitoring a running instance of OpenUnison in your environment.  For that reason, OpenUnison does not come pre-set resources for the operational OpenUnison `Pod`s.  The one exception to this is the operator, which is much more predictable and does have pre-set requests and limits.

## Setting Requests and Limits

You can set requests and limits for the operational components in your values.yaml:

```yaml
services:
  resources:
    requests:
      memory: 1024MiB
      cpu: 2
    limits:
      memory: 1024MiB
      cpu: 2
```

The same limits will be set for all operational containers (openunison-orchestra, kube-oidc-proxy, ouhtml-orchestra-login-portal) and the jobs created by the check-certs `Job`.  They all need about the same memory.  

The test `Pod` used to validate configurations doesn't need nearly as many resources, are set independently at `precheck.resources`.

## Scheduling

THe helm chart values file supports multiple scheduling options.

## Node Selector

A set of node selectors can be set under `services.node_selectors` as a set of name/value pares.  For instance, setting:

```yaml
services:
  node_selectors:
    kubernetes.io/arch: amd64
```

will ensure that OpenUnison's `Pod`s will only be scheduled on amd64 nodes.

## Tolerations

If you want to set a taint on a node that will run OpenUnison `Pod`s, such as to make sure it's running on a control plane specific node, add the tolerations to `services.tolerations`:

```yaml
services:
  tolerations:
  - key: "key1"
    operator: "Exists"
    effect: "NoSchedule"
```

## Pod Affinity

*This feature requires OpenUnison 1.0.42+*

You can configure `Pod` affinity (and anti-affinity) using the [Pod Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity) attributes under `services.affinity`.