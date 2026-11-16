# Air-Gap Implementation

To deploy OpenUnison in an air-gap environment you need to import for the following images into your local repository.  The table notes the image, its function, the chart its referenced in and the Helm chart values to set to customize which image to use.  To get the latest tags, go to the GitHub URL provided in the table below.  

| Image | Function | GitHub URL | Chart | Value Name |
| ----- | -------- | ----- | ---------- | -- |
|ghcr.io/openunison/openunison-kubernetes-operator|OpenUnison Operator|[https://github.com/OpenUnison/openunison-kubernetes-operator/pkgs/container/openunison-kubernetes-operator](https://github.com/OpenUnison/openunison-kubernetes-operator/pkgs/container/openunison-kubernetes-operator)|tremolo/openunison-operator|`operator.image`|
|ghcr.io/tremolosecurity/activemq-docker|ActiveMQ Image Maintained by Tremolo Security|[https://github.com/TremoloSecurity/activemq-docker/pkgs/container/activemq-docker](https://github.com/TremoloSecurity/activemq-docker/pkgs/container/activemq-docker)|tremolo/openunison-*|`amq_image`|
|ghcr.io/openunison/openunison-k8s|Orchestra Image|[https://github.com/OpenUnison/openunison-k8s/pkgs/container/openunison-k8s](https://github.com/OpenUnison/openunison-k8s/pkgs/container/openunison-k8s)|tremolo/orchestra|`image`|
|ghcr.io/tremolosecurity/python-slim-nonroot/python3:|Helm Test Image|[https://github.com/TremoloSecurity/python-slim-nonroot/pkgs/container/python-slim-nonroot%2Fpython3](https://github.com/TremoloSecurity/python-slim-nonroot/pkgs/container/python-slim-nonroot%2Fpython3)|tremolo/orchestra|`openunison.precheck.image`|
|ghcr.io/openunison/openunison-k8s-react|NGINX Image for static HTML & JavaScript|[https://github.com/OpenUnison/openunison-k8s-react/pkgs/container/openunison-k8s-react](https://github.com/OpenUnison/openunison-k8s-react/pkgs/container/openunison-k8s-react)|tremolo/orchestra-login-portal|`openunison.html.image`|

## Rebuilds and Image Scans

These images are scanned using [grype](https://anchore.com/grype/) from Anchore.  If there are any OS level vulnerabilities that have been patched by Canonical, then the images are rebuilt.  
