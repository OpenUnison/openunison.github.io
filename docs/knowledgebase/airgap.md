# Air-Gap Implementation

To deploy OpenUnison in an air-gap environment you need to import for the following images into your local repository.  The table notes the image, its function, the chart its referenced in and the Helm chart values to set to customize which image to use.

| Image | Function | Chart | Value Name |
| ----- | -------- | ----- | ---------- |
|ghcr.io/openunison/openunison-kubernetes-operator|OpenUnison Operator|tremolo/openunison-operator|`image`|
|ghcr.io/tremolosecurity/activemq-docker|ActiveMQ Image Maintained by Tremolo Security|tremolo/openunison-*|`amq_image`|
|ghcr.io/openunison/openunison-k8s:1.0.37|Orchestra Image|tremolo/orchestra|`image`|
|ghcr.io/tremolosecurity/python3|Helm Test Image|`openunison.precheck.image`|
|ghcr.io/openunison/openunison-k8s-html|NGINX Image for static HTML & JavaScript|`openunison.html.image`|
