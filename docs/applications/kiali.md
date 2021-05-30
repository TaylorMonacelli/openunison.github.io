# Kiali

Kiali is the GUI for Istio.  Kiali uses your logged in user's context when communicating with Kubernetes.  When configured with OpenUnison, this means that when a user interacts with Kiali they're only able to see what their user is able to interact with in Kubernetes as limited by the cluster's RBAC policies.  This makes for a better user experience and a more secure implementation.  OpenUnison can provide SSO for it in three ways:

1. OpenID Connect - If you already have Kiali setup behind an Ingress, you can add it as an OIDC client to OpenUnison, passing your user's `id_token` directly to the API server.
2. Reverse Proxy with OpenID Connect - This scenario works the same way as OpenUnison's dashboard integration.  OpenUnison's integrated reverse proxy manages the user's authentication and session, injecting an `id_token` into each request that Kiali will pass to your cluster.
3. Reverse Proxy with Impersonation - Useful on cloud managed clusters where OpenID Connect is not available, this scenario uses OpenUnison's reverse proxy to inject Kubernetes impersonation headers into each request identifying the user to the cluster.

We're quite proud of options two and three, as we developed the contributions that enabled them!

## OpenID Connect

## Reverse Proxy With OpenID Connect

## Reverse Proxy with Impersonation

![Kiali with Impersonation](../../assets/images/kiali_openunison.png)

The user accesses Kiali through the same host name as the access portal.  When the user attempts to access https://network.openunison_host/kiali the user is authenticated, if not already, with their user id and groups added as impersonation headers.  The request is forwarded to Kiali via its `Service` where Kiali will then use the Impersonation headers to interact with the API server using the user's own security context.  Deployment assumes that Kiali is running in the `istio-system` namespace.  Run:

```
kubectl apply -f https://raw.githubusercontent.com/OpenUnison/openunison.github.io/master/docs/assets/yaml/kiali-impersonation.yaml
```

Next, update the Kiali configuration.  Depending on if you are using the operator or have deployed Kiali directly you may need to edit either the `kiali` `ConfigMap` or the `kiali` object.  Which ever configuration you need to update, change `auth.strategy` to `header`.  If you had to edit the `ConfigMap` directly, restart the `kiali` pods.  If you are running the operator, it will restart the pods for you.

Wait a moment and login to OpenUnison, you'll see a new "badge":

![Kiali Badge](../../assets/images/kiali-badge.png)

Clicking on the badge will open Kiali in a new tab. You'll see your username (which is sent to the API server) is in the upper right hand corner:

![Kiali After Login](../../assets/images/kiali-after-login.png)

After logging out of OpenUnison, you will automatically be logged out of Kiali.  