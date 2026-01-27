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

[Return to deployment](../../deployauth#pre-requisites)