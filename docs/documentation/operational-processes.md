# Operational Processes

This page references several operational processes for OpenUnison.

## Force User Logouts / End Sessions

There are situations where a user's session must be terminated immediately.  Depending on OpenUnison's configuration, there are two ways to do this when working with Kubernetes.  You can manually delete the user's OpenID Connect sessions to keep them from refreshing their tokens and create an `EndSession` object to force a user logout from web sessions.

### Kubernetes Login Portal

Using the login portal to end a user's sessions quickly is done manually via either kubectl or directly using the Kubernetes API.

#### Ending OpenID Connect Sessions

OpenUnison uses very short lived tokens, between 1-2 minutes depending on your clock skew, requiring your client or `kubectl` to constantly refresh.  This entails sending a refresh token to OpenUnison to request a new `id_token` and `refresh_token`.  OpenUnison stores this information, encrypted, as CRDs in your cluster.  You can delete these CRDs, meaning that the next time your client tries to refresh the `id_token`, the process will fail and the user will no longer be able to interact with your cluster.

| Step | Explanation | Example |
| ---- | ----------- | ------- |
| Determine the user DN to delete | Each user that's authenticated to OpenUnison has a Distinguished Name (DN) that identifies the user inside of the internal LDAP virtual directory.  This DN will take the form `uid=user,ou=shadow,o=Tremolo` for the login portal,  You can find the exact DN by looking in the OpenUnison access logs.  The DN is in every `AzLog` line. | `uid=mmosley,ou=shadow,o=Tremolo` |
| Generate a sha1 hash of the DN | OpenUnison stores a SHA1 hash of the  user's DN as a label on each session object to make it easier to delete. | `echo -n 'uid=mmosley,ou=shadow,o=Tremolo' | sha1sum` generates `5204c08a79c11dbdc2fdda1e07bc866fb4f22888  -` |
| Delete all instances of the `oidc-session` object with the sha1 label | You can delete the sessions using `kubectl` | `k delete oidc-sessions -l tremolo.io/user-dn=5204c08a79c11dbdc2fdda1e07bc866fb4f22888 -n openunison` |

Once the `oidc-session` objects are deleted, sessions will end once tokens expire.

#### Ending Web Sessions

Prior to 1.0.43, killing the `oidc-sessions` was enough to also end portal and dashboard sessions.  However, with the desire to have more web accessible tools, we separated this by default in 1.0.43.  To kill a user's web session in 1.0.43+, you need to create a special object called `EndSession` with the user's DN.  For instance, the below object:

```yaml
kind: EndSession
apiVersion: openunison.tremolo.io/v1
metadata:
  name: kill-mmosley
  namespace: openunison
spec:
  dn: uid=mmosley,ou=shadow,o=Tremolo
```

Will delete the user `mmosley`.  The DN will be the same as what was used to generate the sha1sum when deleting OIDC sessions.  You don't need to worry about deleting this object, OpenUnison will delete it after about a minute.

### Namespace as a Service (NaaS) Portal

If you've deployed the NaaS portal, you can [delete user's sessions quickly from the portal using the **End sessions** workflow.](/documentation/naas/#end-sessions)

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