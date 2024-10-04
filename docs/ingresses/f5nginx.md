# F5 NGINX Integration

The [F5 NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller) is the Ingress controller provided by F5.  It is ***NOT the NGINX Ingress controller provided by the Kubernetes project.***  The F5 NGINX controller includes both the open source version and ngnixPlus.  To configure:

```yaml
network:
  ingress_type: f5nginx
```

The default `Ingress` `class` is `nginx`.  You can change it by specifying an `Ingress` `class`:

```yaml
network:
  ingress_type: f5nginx
  ingress_class: nginxPlus
```

The open source version of the F5 NGINX Ingress controller doesn't support sticky sessions, so you can only have one OpenUnison replica running.  If you're deploying NGINX Plus, you can enable it by adding the `network.f5nginx.plus` configuration:

```yaml
network:
  ingress_type: f5nginx
  ingress_class: nginxPlus
  f5nginx:
    plus: true
```

## No Load Balancer?

If you don't have a load balancer, run the following to expose NGINX directly via a host port:

```
kubectl patch deployments ingress-nginx-controller -n ingress-nginx -p '{"spec":{"template":{"spec":{"containers":[{"name":"controller","ports":[{"containerPort":80,"hostPort":80,"protocol":"TCP"},{"containerPort":443,"hostPort":443,"protocol":"TCP"}]}]}}}}'
```

[Return to deployment](../../deployauth#pre-requisites)