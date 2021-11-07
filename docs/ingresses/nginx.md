# NGINX Integration

OpenUnison works out-of-the-box with the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/).  Specify `network.ingress_type=nginx` in your values.yaml.  The
helm chart will generate the proper `Ingress` object for you, along with sticky sessions.  You can add annotations to the 
`network.ingress_annotations` object in your values.yaml for additional annotations in your `Ingress` object, such as for using Let's Encrypt certificates from cert-manager.

## No Load Balancer?

If you don't have a load balancer, run the following to expose NGINX directly via a host port:

```
kubectl patch deployments ingress-nginx-controller -n ingress-nginx -p '{"spec":{"template":{"spec":{"containers":[{"name":"controller","ports":[{"containerPort":80,"hostPort":80,"protocol":"TCP"},{"containerPort":443,"hostPort":443,"protocol":"TCP"}]}]}}}}'
```

[Return to deployment](../../deployauth#pre-requisites)