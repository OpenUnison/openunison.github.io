# Argo Workflows

<div style="padding:56.25% 0 0 0;position:relative;"><iframe src="https://player.vimeo.com/video/1040091560?title=0&amp;byline=0&amp;portrait=0&amp;badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" frameborder="0" allow="autoplay; fullscreen; picture-in-picture; clipboard-write; encrypted-media" style="position:absolute;top:0;left:0;width:100%;height:100%;" title="OpenUnison and Argo Workflows"></iframe></div><script src="https://player.vimeo.com/api/player.js"></script>

Argo Workflows is a generic workflow engine that runs on Kubernetes.  While it supports SSO via OpenID Connect, it has an additional requirement to map the logged-in user to a `ServiceAccount` that is authorized to interact with the cluster.  This chart automates the integration with a just-in-time workflow that:

1. Creates a `ServiceAccount` to represent the user
2. Create's a `Secret`, bound to the user's `ServiceAccount` that Argo will use
3. Map's the user's groups to `RoleBinding`s and `ClusterRoleBinding`s to provide identical permissions
4. Provisions the user's `ServiceAccount` to the mapped `RoleBinding`s and `ClusterRoleBinding`s

The first step is to setup OpenUnison, then setup Argo Workflows.

## OpenUnison Setup

First, create a secret with random data for your client secret:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: argowf
  namespace: openunison
data:
  clientsecret: ZG9ub3R1c2UhISEh
```

Next, you'll need to add following values to your openunison values:

```yaml
argowf:
  # The hostname for Argo Workflows
  hostname: argowf.talos.local
  # the namespace where Argo Workflows are installed
  namespace: argowf
  group_map:
    # you can choose to create this ConfigMap manually by specifying false
    create: true
    # list each group and the binding they'll be mapped to
    map:
    - 
      # the name of the group from the user's JWT
      group: "CN=k8s-admins,CN=Users,DC=ent2k22,DC=tremolo,DC=dev"
      # If mapping to a ClusterRoleBinding, specify crb.  If mapping to a RoleBinding, specify rb
      kind: crb
      # the name of the binding
      name: argowf-clusteradmins
      # If a RoleBinding, the Namespace the RoleBinding is in
      namespace: ""
```

Next, update OpenUnison's values to call the `jit-argowf` `Workflow` after authentication:

```yaml
openunison:
  post_jit_workflow: jit-argowf
```

Once updated, rerun the `tremolo/orchestra-login-portal` chart and install the `tremolo/argowf` chart:

```sh
$ helm upgrade orchestra-login-portal tremolo/orchestra-login-portal -n openunison -f /path/to/openunison-values.yaml
$ helm upgrade --install argowf tremolo/argowf -n openunison -f /path/to/openunison-values.yaml
```

When you next login to OpenUnison, a `ServiceAccount` for your user will be created in the Argo Workflow namespace along with the `Secret`.  Also, permissions will be synced based on your mappings.  Next, you can move on to deploying Argo.

## Argo Workflow Setup

The first step is to create your namespace and a `Secret` that will store your OpenID Connect configuration.  Assuming you're deploying Argo to the `argowf` `Namespace`, create a `Secret` with the same secret value that you created in the `openunison` `Namespace` above:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: argowf
  namespace: argowf
data:
  # base64 encoded argowf
  clientid: YXJnb3dm
  # the same value as created earlier in the openunison namespace
  clientsecret: ZG9ub3R1c2UhISEh
```

Finally, deploy Argo Workflow with this minimum values yaml:

```yaml
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - argowf.talos.local
    tls:
    - hosts:
      - argowf.talos.local
  authModes:
  - sso
  sso:
    enabled: true
    issuer: https://k8sou.talos.local/auth/idp/k8sIdp
    clientId:
      name: argowf
      key: clientid
    clientSecret:
      name: argowf
      key: clientsecret
    redirectUrl: https://argowf.talos.local/oauth2/callback
    customGroupClaimName: groups
    insecureSkipVerify: true
```

Change all instances of `argowf.talos.local` to Argo's host name.  Change all instances of `k8sou.talos.local` to the host name for OpenUnison.

## Logging In

Once deployed, login to OpenUnison and click on the Argo Workflows badge:

![OpenUnison with Argo Workflow badge](/assets/images/apps/argowf/openunison-argowf.png)

You'll be logged in to Argo Workflows and go directly to the Workflows page.  If you click on the user info icon, you'll see that you're logged in and your groups:

![Argo Workflow User Info Page](/assets/images/apps/argowf/argowf-userinfo.png)

## Next Steps

You can check out our [blog post that dives into the background on how the integration works](https://www.tremolo.io/post/argo-workflows-sso).  You can also look at automating onboarding via OpenUnison using the Namespace as a Service portal.  