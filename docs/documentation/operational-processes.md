# Operational Processes

This page references several operational processes for OpenUnison.

## Upgrading Satellite Clusters

When using OpenUnison, either as an SSO portal or using NaaS, with a control plane/satellite model there are two ways to upgrade an integrated satellite:

1. Re-run `ouctl install-satelite` to completely upgrade both the satellite's OpenUnison AND the integration with the control plane
2. Manually upgrade the control plane integration

Each control plane has a helm chart deployment that corresponds to each satellite.  For instance, if you run `helm list -n openunison` on your control plane, you'll see something very similar to:


```sh
NAME                            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                           APP VERSION
cluster-management              openunison      73              2025-07-28 14:17:35.930717 -0400 EDT    deployed        openunison-k8s-cluster-management-3.0.56        1.0.42     
openunison                      openunison      117             2025-07-28 14:16:25.645271 -0400 EDT    deployed        openunison-operator-3.0.16                      1.0.42     
openunison-privileged-access    openunison      28              2025-07-28 14:17:46.858205 -0400 EDT    deployed        openunison-privileged-access-1.0.1              1.0.42     
orchestra                       openunison      112             2025-07-28 14:16:27.081721 -0400 EDT    deployed        orchestra-3.1.15                                1.0.42     
orchestra-login-portal          openunison      38              2025-07-28 14:17:33.39816 -0400 EDT     deployed        orchestra-login-portal-2.3.69                   1.0.42     
satellite-XXXX-legacy-1         openunison      5               2025-04-30 19:48:31.328533193 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.11               1.0.42     
satellite-XXXX-legacy-2         openunison      6               2025-05-01 12:27:41.211731231 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.11               1.0.42     
satellite-XXXX-legacy-3x        openunison      10              2025-07-23 18:12:59.675713458 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.11               1.0.42     
satellite-XXXX-legacy-4         openunison      15              2025-05-01 12:41:33.704882516 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.11               1.0.42     
satellite-XXXX-legacy-5         openunison      5               2025-05-01 12:45:52.793441525 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.11               1.0.42     
satellite-sat-10                openunison      21              2025-07-23 14:15:11.837078 -0400 EDT    deployed        openunison-k8s-add-cluster-1.0.12               1.0.42     
satellite-sat5                  openunison      1               2024-07-24 17:51:53.671426764 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.8                1.0.40     
satellite-sat6                  openunison      1               2024-10-28 12:29:57.470035386 +0000 UTC deployed        openunison-k8s-add-cluster-1.0.9                1.0.41 
```

you can see there are a number of satellite clusters that have been integrated.  If you don't want to re-run a full integration, but just want to update the control plane integration to get the latest version of a chart, update permissions, enable a new feature that doesn't require a full re-deployment of a satellite, etc, then you can instead just re-deploy the helm chart.

When you run `ouctl install-satelite`, it will generate a client values.yaml that is used for the `openunison-k8s-add-cluster` chart that is deployed.  If you haven't stored that values file, you can get it directly using helm.  For instance, if we're going to update `satellite-sat-10`:

```sh
$ helm get values satellite-sat-10 -n openunison
USER-SUPPLIED VALUES:
cluster:
  additional_badges:
  - description: Kiali
    icon: iVBORw0KGgoAAA...
    name: kiali
    url: https://ou.192-168-2-10.nip.io/kiali
  az_groups:
  - k8s-namespace-administrators-k8s-sat-10-*
.
.
.
```

The output of this command can be piped to a file, edited, and then used as the values for a helm command.  For instance:

```sh
$ helm get values satellite-sat-10 -n openunison > /path/to/values.yaml
.
.
make your edits
.
.
$ helm upgrade satellite-sat-10 oci://harbor.domain.internal/mycharts/openunison-k8s-add-cluster --version=1.0.14 -n openunison -f /path/to/values.yaml
```

Assuming the `helm upgrade` is successful, OpenUnison will pick up the changes without needing to restart OpenUnison.