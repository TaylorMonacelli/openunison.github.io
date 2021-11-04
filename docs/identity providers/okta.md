# Okta

Okta is most easily integrated via OpenID Connect.  Okta doesn't include `groups` as a claim in the `id_token`, you need to get it from the `user_info` endpoint.  You'll need:

1. client secret
2. client id
3. issuer

This guide will take you step-by-step through configuring Okta and OpenUnison to enable SSO for your cluster.

## Configuring Okta

First, login to your Okta dashboard and click on **SSO Apps**:

![Okta Main Page](../../../assets/images/identity-providers/okta/screen1.png)

Next, click on **Create Application**:

![Okta Applications](../../../assets/images/identity-providers/okta/screen2.png)

The screen will grey-out with a "popup" asking you what kind of application you want to create.  Choose **OIDC - OpenID Connect** for **Sign-in method** and **Web Application** for **Application type**:

![Okta New Application](../../../assets/images/identity-providers/okta/screen3.png)

The next screen will let you name your application and provide a logo.  It will also ask for the callback URL for OpenUnison.  This step is used to ensure that no one tries to force Okta to redirect your authentication to an attacker.  The URL will be the value of *https://****network.openunison_host****/auth/oidc*, where ***network.openunison_host*** comes from your values.yaml.  Since our demo ***network.openunison_host*** is `k8sou.apps.ou.tremolo.dev`, the Sign-in redirect URIs is `https://k8sou.apps.ou.tremolo.dev/auth/oidc`.  At the bottom of the form you can specify which groups have access to this application.  For simplicty we specified **Allow everyone in your organization to access**:

![Okta Application Configuration ](../../../assets/images/identity-providers/okta/screen4.png)

Once you click **Save**, you'll be brought to the final application configuration screen which will have the data you need to configure OpenUnison.

## Configuring OpenUnison

In Okta, navigate to your application if you aren't already there.  You can get the data you need from the screen:

|OpenUnison Field|Configuration Type|Okta Section|Okta Field|
|----------------|------------------|------------|----------|
|oidc.client_id  | helm values.yaml | Client Credentials | Client ID |
|OIDC_CLIENT_SECRET | `orchestra-secrets-source` `Secret` | Client Credentials | Client secret |
|oidc.issuer | helm values.yaml | General Settings | Okta Domain |

![Okta Application ](../../../assets/images/identity-providers/okta/screen5.png)

The above diagram highlights where to get this information from.  Use your **Okta Domain** in the URL for your issuer (don't forget the `https://`!).

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

Once you have updated the `orchestra-secrets-source` with your `OIDC_CLIENT_SECRET` and updated the `oidc` section of your values.yaml, you can continue with the [Deploy the Portal](../../deployauth#deploy-the-portal) step.


## Namespace as a Service

The [Namespace as a Service](../../namespace_as_a_service) portal supports looking up groups in Okta when working with [external group management](../../namespace_as_a_service/#choosing-okta-groups).

## Other Resources

See our [blog post on Okta and Kubernetes integration](https://www.tremolosecurity.com/post/using-okta-with-kubernetes).