# Gateway API

The Kubernetes project has released the Gateway API and is encouraging users to migrate from the Ingress API.  The charts support the Gateway API with some caveats:

* ***No TLS From the Gateway to OpenUnison*** - The current Gateway API specification doesn't support TLS termination and re-encryption.  If using OpenUnison with the Gateway API you are ***HIGHLY*** encouraged to use a service mesh to enable encryption between your gateway and OpenUnison given that OpenUnison is managing tokens and authentication.
* ***No Cookie Based Sticky Sessions*** - Cookie based sticky sessions are an extension that is available for some providers, however it is not standard and we haven't included the objects to support it in this chart.  If your provider supports sticky sessions then you'll want to create those objects on your own.
* ***Impersonation uses TLS Passthrough to kube-oidc-proxy*** - Since re-encryption is not supported and kube-oidc-proxy only runs with TLS, kube-oidc-proxy is configured as a TLS passthrough.

OpenUnison has been tested with Cilium's Gateway API implementation though it should work with any known provider.  In order to enable the gateway API:

```yaml
network:
  force_redirect_to_tls: false
  ingress_type: gatewayapi
  gatewayapi:
    class: cilium

  .
  .
  .
```

## Bring Your Own Gateway

By default, the Helm chart creates a simple `Gateway` that will provide a listener that will decrypt traffic for web components, and TLS passthrough for the kube-oidc-proxy if you're using impersonation.  Here's what that typical `Gateway` will look like:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    cert-manager.io/cluster-issuer: enterprise-ca
    meta.helm.sh/release-name: orchestra
    meta.helm.sh/release-namespace: openunison
  name: openunison-orchestra
  namespace: openunison
spec:
  gatewayClassName: cilium
  listeners:
  - allowedRoutes:
      namespaces:
        from: Same
    hostname: k8sou.192-168-2-13.nip.io
    name: openunison
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: ou-tls-certificate
      mode: Terminate
  - allowedRoutes:
      namespaces:
        from: Same
    hostname: k8sapi.192-168-2-13.nip.io
    name: kube-oidc-proxy
    port: 443
    protocol: TLS
    tls:
      mode: Passthrough
```

To disable the creation of this `Gateway` and provide your own, in your values.yaml:

```yaml
network:
  gatewayapi:
    class: cilium
    gateway:
      # disable the creation of the Gateway object
      create: false
      # specify the parent_refs for the web components of OpenUnison
      parent_refs:
      - name: openunison-orchestra-manual
        namespace: openunison
        sectionName: openunison
      # specify the parent_refs for the kube-oidc-proxy
      kube_oidcproxy_parent_refs:
      - name: openunison-orchestra-manual
        namespace: openunison
        sectionName: kube-oidc-proxy
```

With these updates, the `HTTPRoute` objects will reference the specified `Gateway` instead of creating one.

[Return to deployment](../../deployauth#pre-requisites)