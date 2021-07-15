# Okta

Okta is most easily integrated via OpenID Connect.  Okta doesn't include `groups` as a claim in the `id_token`, you need to get it from the `user_info` endpoint.  You'll need:

1. client secret
2. client id
3. issuer

In Okta, your `issuer` is the URL for your Okta account.  See the below example of an Okta configuration:

```
oidc:
  client_id: XXXXXXXXXX
  issuer: https://yyyyyy.okta.com/
  user_in_idtoken: false
  domain: ""
  scopes: openid email profile groups
  claims:
    sub: sub
    email: email
    given_name: given_name
    family_name: family_name
    display_name: name
    groups: groups
```

See our [blog post on Okta and Kubernetes integration](https://www.tremolosecurity.com/post/using-okta-with-kubernetes) for screenshots.