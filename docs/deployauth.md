# Deploying The Authentication Portal

The authentication portal is the best place to start.  This guide is a step-by-step explanation of each of the components
of the portal, what they do and how to configure them.  

## Upgrading from 1.0.23 and Earlier

You can quickly [upgrade your existing OpenUnison deployment](../../upgrading) without making significant changes to your deployment.

## Deploying the Login Portal

These are the step-by-step instructions for deploying OpenUnison with Kubernetes.  Each step provides some explanation, but with greater details linked out through this document.  The goal of this section is to give you the deployment instructions as succinctly as possible, with supporting details provided in reference sections.  OpenUnison can be deployed in multiple ways and its deployment will vary based on your cluster's configuration and needs.  There are four steps to the deployment are:

| Deployment Phase | Description | Approximate Time |
| ---------------- | ----------- | ---------------- |
| Pre-requisites   | Deploy the `Ingress` controller and dashboard. | This is dependent on how long it takes to deploy and validate your `Ingress` controller of choice and accompanying network infrastructure such as load balancers. |
| Site Specific Configuration | This is where you'll configure OpenUnison for your authentication source and for your infrastructure by configuring a Helm chart values.yaml and generating your configuration `Secret` | Generally 5 - 30 minutes depending on if all your prerequisite are ready |
| Deploy the Portal | Use the `ouctl` tool to deploy your portal | Less then 5 minutes |
| *Optional* - Integrating Your Cluster | If you're using OpenID Connect to integrate directly with your cluster, you'll need to configure your cluster to trust OpenUnison for authentication | Dependent on the Kubernetes distribution, usually less then five minutes |

In any OpenUnison deployment the most amount of time spent is on getting the networking for your cluster working and getting the right authentication configuration for your identity store.  Once those are ready the rest of the deployment is very direct.

### Pre-requisites

**Ingress Controller**

In order to use OpenUnison, you'll need to first have an `Ingress` controller deployed.  While any controller should work, we have documented deployment steps for these controllers.  If you need help deploying an additional type of controller, please open an [issue](https://github.com/openunison/openunison-k8s/issues) and we'll be happy to help.

| Ingress Controller | 
| ------------------ | 
| [NGINX](../ingresses/nginx) |
| [Istio](../ingresses/istio) |

Once your `Ingress` controller is deployed, the next step is to deploy the dashboard.


**The Kubernetes Dashboard**

The [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) is a powerful and simple way to work with your cluster without having access to a command line.  It is accessed securely by using the user's own permissions, with the dashboard its self having no permissions in your cluster.  OpenUnison manages the login process for you, so there's no need to upload a kubectl configuration to the dashboard to make it work.  The dashboard can be deployed with:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
```

***DO NOT SETUP AN INGRESS FOR YOUR DASHBOARD*** - OpenUnison takes care of that for you.

Having deployed the dashboard, next we'll deploy the base configuration for OpenUnison.



### Site Specific Configuration

This section is where you'll do the work of configuring OpenUnison for your cluster and spend the most time.  The deployment steps are here, with links to detailed configuration options to guide you through the process.

First, get the latest [default values.yaml](/assets/yaml/openunison-default.yaml) and customize it.  There are three minimum configuration sections you need to address:

| Values Section | Decision Points | Notes |
| -------------- | --------------- | ----- |
| `enable_impersonation` | Determine if you want to integrate your cluster directly with OpenUnison using OpenID Connect (`false`), or use OpenUnison's integrated impersonating reverse proxy when interacting with the API server (`true`).  In general, if you're working with an on-premises cluster this will be `false`.  If you're using a hosted, or managed, cluster such as EKS or Civo this would by `true`.  |
| `network`      | This section describes how you will access OpenUnison and how OpenUnison will interact with your cluster | [Host Names and Networking](#host-names-and-networking) |
| Authentication (***One*** of `activedirectory`, `oidc`, `github`, or `saml`) | How will OpenUnison authenticate users?  This is covered in detail next.  At least one option is required. | [Choosing an Identity Source](#choosing-an-identity-source) |

***NOTE***: While not required, you should update the `k8s_cluster_name` value in your values.yaml to have a descriptive label for your cluster.

Once you've chosen an identity source, return here to finish the installation.

### Deploy the Portal

OpenUnison is deployed using a series of Helm Charts that cover different aspects of the deployment between configuration, integration with the API server, and the operator that manages certificates for you.  You can [deploy these charts manually](#manual-deployment), but the `ouctl` command makes this much easier.  First, download `ouctl` for the correct platform:

* [Linux](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.8-linux)
* [Windows](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.8-win.exe)
* [MacOS](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.8-macos)

Rename the downloaded file to `ouctl` (or `ouctl.exe` on windows).  Next, download the OpenUnison helm repo:

```
helm repo add tremolo https://nexus.tremolo.io/repository/helm/
helm repo update
```

The next step is to create a file with your secret in it for AD/LDAP, OIDC, or GitHub login.  

---
**SECURITY NOTE**

You shouldn't use the `echo` command to create this file for two important reasons:

1. It adds an extra `\n` at the end
2. It will store your secret information in your shell's history file

Best to create this file manually and add the secret to it. ***NOTE***: Do not base64 encode the data before putting it in this file.  The `ouctl` command will handle that for you.

---

Finally, deploy the portal:

```
ouctl install-auth-portal -s /path/to/secret/file /path/to/yaml
```

Assuming there are no issues, OpenUnison will be deployed and ready for access.  The `ouctl` command is safe to re-run.  If you want to update the secret, you can provide the `-s` again.  If you just want to upgrade the charts, you can run `ouctl` without `-s` and it will just update the charts.

If you're going to integrate your cluster with OpenID Connect (most on-prem clusters will be integrated this way), the final step is to enable SSO with your Kubernetes cluster. If you configured enable_impersonation to true, skip this step.

Finally, login to your portal by going to https://`network.openunison_host`/, where `network.openunison_host` is the host name you specified in your values.yaml. If everything was setup correctly, you can now start working with your cluster!

Learn to use the OpenUnison Login Portal by exploring our [user guide](../documentation/login-portal).

### Integrating Your Kubernetes Cluster

If you're running on a [managed cluster](#managed-clusters) such as EKS, AKS, or GKE, you can skip this section.

To get the correct flags for your API server, run

```
kubectl describe configmap api-server-config -n openunison
```

If you're using the certificate generated by OpenUnison, you can retrieve it with:

```
kubectl get secret ou-tls-certificate -n openunison -o json | jq -r '.data["tls.crt"]' | base64 -d > /path/to/ou-ca.pem
```

Next, depending on your distribution, update your cluster's command line parameters with the output for describing the above `ConfigMap`.
If you do not need to explicitly trust a certificate, skip `--oidc-ca-file`.  Once you're API servers restart, you should be able to use
either the dashboard or the kubectl API.

## Choosing an Identity Source

Before starting the deployment process, choose how you want to authenticate your users.  This is how OpenUnison will authenticate users,
regardless of how OpenUnison integrates with your cluster.  Each authentication options includes prerequisite that must be collected
before deployment.  Below are the available options:

| Option | Notes |
| ------ | ----- |
| [Active Directory / LDAP](#active-directory-ldap) | Popular with enterprises and on-premises deployments |
| [OpenID Connect](#openid-connect) | Most common integration method.  Useful with common providers such as Okta, Azure Active Directory, and Google. |
| [GitHub](#github) | Useful for when you want to drive management based on your GitHub organizations and teams |
| [SAML2](#saml2) | Most often used in legacy environments with Microsoft's Active Directory Federation Services (ADFS) |

At the end of each section is a link to bring you back to the deployment.

### Active Directory / LDAP

Using Active Directory or LDAP, OpenUnison collects your user's username and password then uses an LDAP bind operation against your Active Directory or LDAP server.  This is
common in many enterprises.  The main advantage to this approach is it's simplicity.  It does however require that the user's password move
through OpenUnison (and your cluster).

![OpenUnison and Active Directory](assets/images/ou-auth-ad.png)

To get started, you will need:

1. The full distinguished name and password for a service account.  This account needs no privileges.  The distinguished name will look something like `cn=serviceaccount,cn=Users,dc=domain,dc=com`
2. The host and port for your domain controllers OR the domain that you want to query.  OpenUnison doesn't know how to talk to multiple hosts, so you will either need to keep your domain controllers behind a load balancer or if your cluster is connected via DNS to the 
domain OpenUnison can use SRV records to discover domain controllers
3. The CA certificate for your domain controllers

Your domain controllers **MUST** have TLS enabled and running.  Most Active Directory deployments will not allow authentication over
a plain text connection.

The following attributes must be available to OpenUnison:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| samAccountName | The user's login ID | mmosley |
| memberOf | The DN of all groups the user is a member of | cn=group1,cn=users,dc=domain,dc=com |

Additionally, users may also have:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| givenName | User's first name | Matt |
| sn | User's last name | Mossley |
| mail | User's email address | mmosley@tremolo.dev |
| displayName | The user's display name | Matt Mosley |

Any of these attribute names can be changed by [customizing the MyVirtualDirectory configuration](../knowledgebase/customizingmyvd) used by 
OpenUnison.

Once all the prerequisites have been gathered, uncomment the `activedirectory` section of your values.yaml and update accordingly:

```
active_directory:
  base: cn=users,dc=ent2k12,dc=domain,dc=com
  host: "192.168.2.75"
  port: "636"
  bind_dn: "cn=Administrator,cn=users,dc=ent2k12,dc=domain,dc=com"
  con_type: ldaps
  srv_dns: "false"
```

The detailed explanation of each setting is below:

| Property | Description |
| -------- | ----------- |
| active_directory.base | The search base for Active Directory |
| active_directory.host | The host name for a domain controller or VIP.  If using SRV records to determine hosts, this should be the fully qualified domain name of the domain |
| active_directory.port | The port to communicate with Active Directory |
| active_directory.bind_dn | The full distinguished name (DN) of a read-only service account for working with Active Directory |
| active_directory.con_type | `ldaps` for secure, `ldap` for plain text |
| active_directory.srv_dns | If `true`, OpenUnison will lookup domain controllers by the domain's SRV DNS record |

The secret file used with the `ouctl` command will contain the password for your service account.  If configuring OpenUnison manually, update the `orchestra-secrets-source` `Secret` with a key called `AD_BIND_PASSWORD` that contains the base64 encoded password for the service account named
in `activedirectory.bind_dn`.

#### RBAC Bindings

If you can populate groups in Active Directory for Kubernetes, you can use those groups for authorization via OpenUnison.  OpenUnison will provide all of a user's groups via the `id_token` supplied to Kubernetes.  The `groups` claim is a list of values, in this case the Distinguished Names of the user's groups.  As an example, I created a group in AD called `k8s_login_ckuster_admins` in the `Users` container of my `ent2k12.domain.com` domain.  This means the group will be `CN=k8s_login_ckuster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com` (you can get the exact name of the group from the `distinguishedName` attribute of the group in Active Directory).  To authorize members of this group to be cluster administrators, we create a `ClusterRoleBinding`:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: activedirectory-cluster-admins
subjects:
- kind: Group
  name: CN=k8s_login_ckuster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Once completed, continue to [Deploy the Portal](#deploy-the-portal) to finish the OpenUnison deployment.

### OpenID Connect

The OpenID Connect protocol is a popular protocol used by Google, Okta, and many others.  It is a well documented standard that is usually the first choice of most
SSO implementations.  While Kubernetes can integrate directly with most OpenID Connect providers, those providers won't generate a kubectl configuration, support the dashboard, or support other cluster management applications.  OpenUnison will support this for your without having to make multiple integrations with your identity
provider.

These following steps will work with any OpenID Connect compatible identity provider.  Here are detailed instructions for specific implementations:

* [Okta](../../identity%20providers/okta)
* [Keycloak](../../identity%20providers/keycloak)
* [AzureAD](../../identity%20providers/azuread)


![OpenUnison and OpenID Connect](assets/images/ou-auth-oidc.png)

To get started, you'll need:

1. Your identity provider's issuer
2. A client id
3. A client secret
4. Claims from your identity provider

The following are required for OpenUnison:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| sub | The user's unique id | mmosley |
| groups | The DN of all groups the user is a member of | admins,users |

Additionally, users may also have:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| given_name | User's first name | Matt |
| family_name | User's last name | Mossley |
| email | User's email address | mmosley@tremolo.dev |
| name | The user's display name | Matt Mosley |

You'll need to provide a redirect URL to your identity provider.  The URL will be `https://[network.openunison_host]/auth/oidc`.  For instance if your `network.openunison_host` 
is k8sou.domain.com your direct URL is `https://k8sou.domain.com/auth/oidc`.

For previously tested identity providers and how to configure them, see the [Identity Providers](../identity%20providers) section of the documentation.

To configure OpenID Connect for authentication to your Kubernetes cluster, uncomment the `oidc` section of the base values.yaml file:

```
oidc:
  client_id: abcdefgh
  issuer: https://dev-XXXX.okta.com/
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

You can customize these values based on your identity provider's needs.  

| Property | Description |
| -------- | ----------- |
| oidc.client_id | The client ID registered with your identity provider |
| oidc.auth_url | Your identity provider's authorization url |
| oidc.token_url | Your identity provider's token url |
| oidc.domain | An email domain to limit access to |
| oidc.user_in_idtoken | Set to `true` if the user's attributes (such as name and groups), is contained in the user's `id_token`.  Set to `false` if a call to the identity provider's user info endpoint is required to load the full profile |
| oidc.userinfo_url | If `oidc.user_in_idtoken` is `false`, the `user_info` endpoint for your identity provider |
| oidc.scopes | The list of scopes to include, may change based on your identity provider |
| oidc.claims.sub | If specified, the claim from the `id_token` to use for the `sub` attribute |
| oidc.claims.email | If specified, the claim from the `id_token` to use for the `mail` attribute |
| oidc.claims.givenName | If specified, the claim from the `id_token` to use for the `givenName` attribute |
| oidc.claims.familyName | If specified, the claim from the `id_token` to use for the `sn` attribute |
| oidc.claims.displayName | If specified, the claim from the `id_token` to use for the `dipslayName` attribute |
| oidc.claims.groups | If specified, the claim from the `id_token` to use for the `groups` attribute |

If you prefer to keep your identity provider's `client_id` as a `Secret`, store it as `OIDC_CLIENT_ID` in the `orchestra-secrets-source` `Secret` in the `openunison` namespace and set `oidc.client_id_is_secret: true` in your values.yaml.  The `client_id` will be read from the `Secret` instead of being stored directly in the configuration object.

The secret file used with the `ouctl` command will contain the client secret.  If you want to configure OpenUnison manually, add `OIDC_CLIENT_SECRET` to the `orchestra-secrets-source` `Secret` in the `openunison` namespace with the base64 encoded client secret from your identity provider.

***If using `echo DATA | base64` to encode your `Secret`, make sure to use `-n` to remove the excess carriage return, ie `echo -n DATA | base64`.  Your authentication will fail otherwise.***

#### RBAC Bindings

The `groups` claim will be passed to Kubernetes to support RBAC bindings.  You can create `RoleBinding` and `ClusterRoleBinding` objects based on this attributes.  For
example if you have a group called `k8s-admins` you can make those users with that group a cluster admin with:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: activedirectory-cluster-admins
subjects:
- kind: Group
  name: k8s-admins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Once completed, continue to [Deploy the Portal](#deploy-the-portal) to finish the OpenUnison deployment.

### GitHub

GitHub is one of the most popular places to store directory your code.  It's also got a powerful identity system that supports SSO and organizing
access to your code.  OpenUnison can leverage this system to provide access to your Kubernetes cluster, using your teams and memberships in RBAC bindings.

![OpenUnison and Active Directory](assets/images/ou-auth-github.png)

To get started, you'll need to create an OAuth app under your organization's `Developer Settings`.  You'll need to provide an `Authorization callback URL`,
which is `https://[network.openunison_host]/auth/github`.  For instance if your `network.openunison_host` is k8sou.domain.com your callback URL is 
`https://k8sou.domain.com/auth/github`.

Once created, you'll need your client id and a client secret.  The client id is added with a list of authorized teams to your values.yaml:

```
github:
  client_id: abcd123
  teams: TremoloSecurity/
```

It's very important to specify at least one team, otherwise anyone on GitHub will have access to your cluster!  GitHub doesn't provide any authorization
for your OAuth apps, that's left to you.  Membership is specified in the form `Organization/Team` where `Organization/` means all teams in that organization.
You can specify multiple teams by commas.

| Property | Description |
| -------- | ----------- |
| github.client_id | The client id from your OAuth2 application configuration |
| github.teams | A comma separated list of authorized teams and organizations.  An organization is listed in the format `OrgName/` and a team in the format `OrgName/TeamName` |

The secret file used with the `ouctl` command will contain the GitHub client secret.  If configuring OpenUnison manually, add `GITHUB_SECRET_ID` to the `orchestra-secrets-source` `Secret` in the `openunison` namespace with the base64 encoded client secret from your OAuth app.

#### RBAC Bindings

Kubernetes will see your user's organizations and teams as groups.  To authorize users based on these groups, list them in your RBAC policies as groups with organizations being in the format `OrgName/` and teams being the format `OrgName/TeamName`.   To authorize members of team `TremnoloSecurity/Owners` to be cluster administrators, we create a `ClusterRoleBinding`:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: github-cluster-admins
subjects:
- kind: Group
  name: TremoloSecurity/Owners
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Once completed, continue to [Deploy the Portal](#deploy-the-portal) to finish the OpenUnison deployment.

### SAML2

SAML2 is a popular user federation protocol, especially in enterprises.  OpenUnison can integrate with your enterprise's SAML2 identity provider for users and
groups.  If you want to test with a SAML2 identity provider, but don't have one you can use Tremolo Security's [testing identity provider](https://portal.apps.tremolo.io/) (requires registration).

To get started, you'll need either the SAML2 metadata URL for your identity provider or the XML of your SAML2 identity provider.  For instructions for specific identity
providers, see the *Identity providers* section of the documentation.

To deploy OpenUnison with SAML2 support, uncomment the `saml2` section of the base values.yaml:

```
saml:
  idp_url: "https://portal.apps.tremolo.io/idp-test/metadata/XXXXXXXXX"
```

Here are the options for the `saml` section:

| Property | Description |
| -------- | ----------- |
| saml.idp_url | The url to your identity provider's saml2 metadata.  If not using a metadata url, set this to an empty string `""` |
| saml.metadata_xml_b64 | Base64 encoded metadata.  Will only be used if `idp_url` is an empty string |

Your identity provider must supply the following attributes:

The following attributes must be available to OpenUnison:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| uid | The user's login ID | mmosley |
| memberOf | The DN of all groups the user is a member of | cn=group1,cn=users,dc=domain,dc=com |

Additionally, users may also have:

| Attribute | Description | Example |
| --------- | ----------- | ------- |
| givenName | User's first name | Matt |
| sn | User's last name | Mossley |
| mail | User's email address | mmosley@tremolo.dev |
| displayName | The user's display name | Matt Mosley |

To stand up your identity provider, you'll need to use OpenUnison's saml2 metadata URL.  The url is https://network.openunison_host/auth/forms/saml2_rp_metadata.jsp. 
For instance if your `network.openunison_host` is k8sou.domain.com the metadata URL will be `https://k8sou.domain.com/auth/forms/saml2_rp_metadata.jsp`.

There is no client secret with SAML2, so you'll skip the `-s` parameter with the `ouctl` command.

#### RBAC Bindings

The `memberOf` attribute from your assertions will be included as the `groups` claim in your `id_token`.  You can use any value from the `memberOf` attribute in
your RBAC bindings.

Once completed, continue to [Deploy the Portal](#deploy-the-portal) to finish the OpenUnison deployment.



## Detailed Configuration

### Host Names and Networking

OpenUnison is a web application that can host multiple systems routed by host name.  If you ever worked with web servers,
this is similar to how virtual sites work.  When OpenUnison receives a request, it routes the request based on it's internal
configuration to the right application using the request's host name and URI path.  To prepare for using OpenUnison to 
manage access to your cluster, you'll need at least two host names:

1. Your OpenUnison portal host name
2. Your K8s Dashboard

If you're running OpenUnison on a managed cluster like EKS or GKE, you'll need a third host name specifically for the api service
that kubectl will use.  

***All of these host names must be different***

This diagram shows the relationships between the host names used to access OpenUnison, your `Ingress` controller, and the load balancer.  In this example, `https://k8sou.domain.com` is the URL for the OpenUnison portal.  The dashboard is accessed using `https://k8sdb.domain.com`, and the API server is accessed using `https://k8sapi.domain.com`.  All three URL's host names point to the same IP address (`a.b.c.d`), which is assigned to the load balancer.

![OpenUnison Network](assets/images/openunison_k8s_network.png)

When your browser goes to `https://k8sou.domain.com/` it's going to a load balancer that forwards the traffic to your `Ingress`
controller.  Depending on your setup your load balancer may be a TLS termination point.  The `Ingress` controller is a TLS termination 
point that decrypts your request and forwards it to the `openunison-orchestra` `Service` which then forwards the packets to the
`openunison-orchestra` `Pod`.  The Helm chart sets all this up for you, so don't worry about having to wire everything yourself.

The dashboard works in a similar way.  Going to `https://k8sdb.domain.com/` goes through the same process as above.  Authentication
works by creating an OpenID Connect trust with `https://k8sou.domain.com/` to enable SSO.  This is why you don't need to setup multiple
connections to your identity system, OpenUnison has centralized authentication.  Once OpenUnison has authenticated you, it forwards
all requests to the dashboard with either your `id_token` or an [Impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
header with your user's information.  Your dashboard runs without privileges, and without it's own `Ingress` object.  All traffic
runs through OpenUnison's reverse proxy.

When using OpenUnison with a managed cluster, such as EKS, you can enable impersonation for authentication to your cluster.
This scenario replaces your API server's host with the value from `network.api_server_host` routing all API requests through
OpenUnison.  OpenUnison authenticates your `id_token` and then injects Impersonation headers into each request.  This allows
you to authenticate to your cluster without getting cloud IAM credentials.  This process will be transparent to your tools, such
as kubectl.

#### No DNS Server?

*If you don't have a DNS server where you can setup host names, try using nip.io.  This service will take any host name with an IP and return the IP address for DNS resolution.  For instance k8sou.apps.192-168-2-119.nip.io will resolve to 192.168.2.119.*

#### No Load Balancer?

It's not uncommon for development releases to not have a load balancer available.  You can configure your `Ingress` controller to listen directly on ports 80 and 443.  Each [Ingress controller's](../ingresses) documentation provides instructions on how to run the controller this way.

#### Network Configuration Options

| Property | Description |
| -------- | ----------- |
| network.openunison_host | The host name for OpenUnison.  This is what user's will put into their browser to login to Kubernetes |
| network.dashboard_host | The host name for the dashboard.  This is what users will put into the browser to access to the dashboard. **NOTE:** `network.openunison_host` and `network.dashboard_host` Both `network.openunison_host` and `network.dashboard_host` **MUST** point to OpenUnison |
| network.api_server_host | The host name to use for the api server reverse proxy.  This is what `kubectl` will interact with to access your cluster. **NOTE:** `network.openunison_host` and `network.dashboard_host` |
| network.k8s_url | The URL for the Kubernetes API server | 
| network.session_inactivity_timeout_seconds | The number of seconds of inactivity before the session is terminated, also the length of the refresh token's session |
| network.createIngressCertificate | If true (default), the operator will create a self signed Ingress certificate.  Set to false if using an existing certificate or LetsEncrypt |
| network.force_redirect_to_tls | If `true`, all traffic that reaches OpenUnison over HTTP will be redirected to HTTPS.  Defaults to `true`.  Set to `false` when using an external TLS termination point, such as an istio sidecar proxy |
| network.ingress_type | The type of `Ingress` object to create.  `nginx` and [istio](https://openunison.github.io/ingresses/istio/) is supported |
| network.ingress_annotations | Annotations to add to the `Ingress` object |
| network.ingress_certificate | The certificate that the `Ingress` object should reference |
| network.istio.selectors | Labels that the istio `Gateway` object will be applied to.  Default is `istio: ingressgateway` |

Return to [Site Specific Configuration](#site-specific-configuration)

### Network Policies

The `orchestra` helm chart will create `NetworkPolicy` objects to lock down all network access to the OpenUnison pods, limiting it to:

* Access from the `Ingress` controller
* Access from within the `openunison` namespace
* Access from the monitoring system's namespace
* Access from the API server

Limiting access to the OpenUnison pods can be important, especially when running OpenUnison in impersonation mode, to cut down attacks from inside
the cluster.  Access from the API server is needed because there are validating webhooks hosted on OpenUnison for validating configuration objects
that are created.  

Enable the creation of policies by setting `network_policies.enabled` to `true`.  This will disable all inbound access to OpenUnison pods from outside
of the OpenUnison namespace.  Then enable which namespaces you want to create ingress `NetworkPolicy` objects for: 

```
network_policies:
  enabled: true
  ingress:
    enabled: true
    labels:
      app.kubernetes.io/name: ingress-nginx
  monitoring:
    enabled: true
    labels:
      app.kubernetes.io/name: monitoring
  apiserver:
    enabled: true
    labels:
      app.kubernetes.io/name: kube-system
```

### Managed Clusters

Most hosted clusters do not support configuring an OpenID Connect identity provider.  The easiest way to manage external access to these clusters is by
enabling `Impersonation` in OpenUnison.  Using `Impersonation`, OpenUnison provides a reverse proxy in front of your API server, enabling external authentication
without having to configure an OpenID Connect identity provider.  To enable impersonation, set `enable_impersonation` to `true` in your values.yaml and make sure
to include a host name for `openunison.api_server_host`. 

#### kubectl exec/cp/port-forward

The kubectl exec, cp, proxy, and port-forward commands use a protocol called SPDY.  This protocol was deprecated by Google in 2016.
Very few modern systems still support SPDY.  If using OpenUnison with a managed cluster like EKS, GKE, or AKS, OpenUnison supports SPDY by integrating [JetStack's OIDC Proxy](https://github.com/jetstack/kube-oidc-proxy). 

```
impersonation:
  use_jetstack: true
  jetstack_oidc_proxy_image: docker.io/tremolosecurity/kube-oidc-proxy:latest
  explicit_certificate_trust: true
```
Here are the details configuration options for impersonation:

| Property | Description |
| -------- | ----------- |
| impersonation.use_jetstack | if `true`, the operator will deploy an instance of JetStack's OIDC Proxy (https://github.com/jetstack/kube-oidc-proxy).  Default is `false` |
| impersonation.jetstack_oidc_proxy_image | The name of the image to use |
| impersonation.explicit_certificate_trust | If `true`, oidc-proxy will explicitly trust the `tls.crt` key of the `Secret` named in `impersonation.ca_secret_name`.  Defaults to `true` |
| impersonation.ca_secret_name | If `impersonation.explicit_certificate_trust` is `true`, the name of the tls `Secret` that stores the certificate for OpenUnison that the oidc proxy needs to trust.  Defaults to `ou-tls-secret` |
| impersonation.resources.requests.memory | Memory requested by oidc proxy |
| impersonation.resources.requests.cpu | CPU requested by oidc proxy |
| impersonation.resources.limits.memory | Maximum memory allocated to oidc proxy |
| impersonation.resources.limits.cpu | Maximum CPU allocated to oidc proxy |

The terminal in the dashboard will work without issue, since the dashboard uses websockets instead of SPDY.

### OpenUnison Resource Configuration

The helm chart can be used to customize openunison's resource utilization and limits and manage access to tokens for communicating with the API server.  You can
also specif node selector labels to limit which nodes your pods run on.

#### TokenRequest API

Starting in Kubernetes 1.20 all tokens mounted into pods are projected TokenRequest API tokens.  These tokens are only valid for the life of the pod
and have an expiration (though it is a year into the future).  OpenUnison can be configured to use shorter lived tokens.  When using the shorter lived
tokens, OpenUnison will reload them when the tokens expire.

```
services:
  enable_tokenrequest: true
  token_request_audience: "https://kubernetes.default.svc.cluster.local"
  token_request_expiration_seconds: 600
  node_selectors: []
```

Here are the detailed configuration options:

| Property | Description |
| -------- | ----------- |
| services.enable_tokenrequest | If `true`, the OpenUnison `Deployment` will use the `TokenRequest` API instead of static `ServiceAccount` tokens. |
| services.token_request_audience | The audience expected by the API server |
| services.token_request_expiration_seconds | The number of seconds TokenRequest tokens should be valid for, minimum 600 seconds | 
| services.node_selectors | annotations to use when choosing nodes to run OpenUnison, maps to the `Deployment` `nodeSelector` |
| services.pullSecret | The name of the `Secret` that stores the pull secret for pulling the OpenUnison image |
| services.resources.requests.memory | Memory requested by OpenUnison |
| services.resources.requests.cpu | CPU requested by OpenUnison |
| services.resources.limits.memory | Maximum memory allocated to OpenUnison |
| services.resources.limits.cpu | Maximum CPU allocated to OpenUnison |

### Customizing The OpenUnison Object

You can customize additions to the `openunison` object generated by the `orchestra` helm chart.  This section lets you specify additional environment
properties for OpenUnison, secrets to load, and specify a custom HTML image. 

```
openunison:
  replicas: 1
  non_secret_data:
    K8S_DB_SSO: oidc
    PROMETHEUS_SERVICE_ACCOUNT: system:serviceaccount:monitoring:prometheus-k8s
    SHOW_PORTAL_ORGS: "true"
  secrets: []
  html:
    image: docker.lab.tremolo.dev/nginx
  enable_provisioning: false
  az_groups: []
```

#### Authorizing Access To Your Cluster

The default implementation of OpenUnison authorizes anyone to authenticate to your cluster and leaves authorization to your RBAC objects.  This
may not be the desired behavior if you know that only users in a certain group should have access to a cluster at all.  For example, if a
cluster was built for the payments team, then developers on the marketing application shouldn't have access to your cluster at all.

To authorize access to the portal, and your cluster, add the authorized groups to `openunison.az_groups` in your values.yaml.  The group names
will be the same as how you would specify them in an RBAC binding. 


#### Detailed configuration options:

| Property | Description |
| -------- | ----------- |
| openunison.replicas | The number of OpenUnison replicas to run, defaults to 1 |
| openunison.non_secret_data | Add additional non-secret configuration options, added to the `non_secret_data` section of the `OpenUnison` object |
| openunison.secrets | Add additional keys from the `orchestra-secrets-source` `Secret` |
| openunison.html.image | A custom NGINX image for the HTML frontend for OpenUnison's apis. |
| openunison.enable_provisioning | If `true`, enables openunison's provisioning and audit database. |
| openunison.az_groups | List of groups to authorize access for this cluster |

## Integrating Other Applications

OpenUnison can support the authentication for your entire environment, not just Kubernetes!  Take a look at our [guides for well known applications](../applications).  Also check out our documentation for [SSO application integration](../documentation/custom-sso).

## Alternate Deployment Methods

### Manual Deployment

#### Base Configuration

The base configuration deploys the groundwork for running OpenUnison.  OpenUnison is a powerful authentication and automation platform that can be used with all of your cluster applications and APIs.  We wanted to make this complexity as easy as possible to manage so we encoded most of the deployment and validation steps in automation technologies.  You'll need [helm version 3+](https://helm.sh/docs/intro/install/) to finish the deployment.

**Create the OpenUnison namespace:**

First, create the `Namespace` that will host OpenUnison.  If using [Istio](../ingresses/istio) or another `Ingress` controller that requires the `Namespace` to be labeled, now is a good time to apply those labels.


```
kubectl create ns openunison
```

**Setup the helm repo with the charts:**

Add OpenUnison's Helm repos to your system.  We host these repos on our own services behind our certificate.

```
helm repo add tremolo https://nexus.tremolo.io/repository/helm/
helm repo update
```

**Install the operator**

Next, install the operator.  The operator is responsible for generating OpenUnison's configuration and consolidating certificates and keys into a central keystore used by OpenUnison.  You can [customize the operator's chart](../documentation/operator) for your environment, but most deployments can use the defaults.

```
helm install openunison tremolo/openunison-operator --namespace openunison
```

Wait until the operator pod is running.  

```
while [[ $(kubectl get pods -l app=openunison-operator -n openunison -o 'jsonpath={..status.conditions[?(@.type=="Ready")].status}') != "True" ]]; do echo "waiting for operator pod" && sleep 1; done
```

**Create a `Secret` that will be used for storing secret information like passwords and shared secrets.**  

OpenUnison separates secret information out of it's configurations.  No secret data should ever be stored in a Helm chart.  A `Secret` object needs to be created to store OpenUnison's secret data (such as passwords, keys, and tokens).  The operator will pull this `Secret` in when generating OpenUnison's configuration.

The below example uses random data.  Do **NOT** use this exact `Secret`, create your own random data for the values
that don't contain an `&`.  A password generator is a great way to generate this data.

Additionally, each authentication method will require its own secret data so this `Secret` will need to be updated.

```
apiVersion: v1
type: Opaque
metadata:
  name: orchestra-secrets-source
  namespace: openunison
data:
  K8S_DB_SECRET: UTJhQzVNNjBMbWljc3Y0UEphYVhsWlRRbw==
  unisonKeystorePassword: SGRVSEg1c1Z0ZUdaTjFxdjJEUHlnYk1wZQ==
kind: Secret
```

#### Deploy the Portal

**Install the orchestra helm chart**

First, install the `orchestra` chart, which does the configuration check and starts the openunison `Pod`.

```
helm install orchestra tremolo/orchestra --namespace openunison -f /path/to/values.yaml
```

If this deployment fails, it's likely from a misconfiguration of your values.yaml.  See [troubleshooting your orchestra deployment failure](../knowledgebase/orchestra_deployment_failed) for instructions on how to debug.

Wait until the orchestra pods are running.  There are validating webhook configurations that will fail in the last step if we 
don't wait.

```
while [[ $(kubectl get pods -l app=openunison-orchestra -n openunison -o 'jsonpath={..status.conditions[?(@.type=="Ready")].status}') != "True" ]]; do echo "waiting for pod" && sleep 1; done
```

**Deploy Portal Configuration**

Last step, deploy the orchestra login portal chart with the ***same values.yaml as the previous chart***:

```
helm install orchestra-login-portal tremolo/orchestra-login-portal --namespace openunison -f /path/to/values.yaml
```

If you're going to integrate your cluster with OpenID Connect (most on-prem clusters will be integrated this way), the final step is to
[enable SSO with your Kubernetes cluster](#integrating-your-kubernetes-cluster).  If you configured `enable_impersonation` to `true`, skip this step.

Finally, login to your portal by going to https://`network.openunison_host`/, where `network.openunison_host` is the host name you specified in your values.yaml.  If everything was setup correctly, you can now start working with your cluster!

Learn to use the OpenUnison Login Portal by exploring our [user guide](../../user-manuals/login-portal).
