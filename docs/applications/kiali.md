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

The user accesses Kiali through the same host name as the access portal.  When the user attempts to access https://network.openunison_host/kiali the user is authenticated, if not already, with their user id and groups added as impersonation headers.  The request is forwarded to Kiali via its `Service` where Kiali will then use the Impersonation headers to interact with the API server using the user's own security context.