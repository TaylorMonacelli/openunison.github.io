# Namespace as a Service

Once you have authentication working for your cluster, often the next question is "How do I provision workloads?".
A simple way to get started is to provide *Namespace as a Service*, or *NaaS*, so your users can request their
own namespaces be created and let them manage access as needed.  The login portal you've created can be extended
to support self service NaaS portal where `Namespaces`, `RoleBindings`, and other management objects like 
default `NetowrkPolicy` and `ResourceQuota` objects can be created automatically.  Default labels and annotations
can also be added to support your reporting needs.

## Deploying The NaaS Portal

## Before You Deploy

Before moving forward, you will need two additional components are required:

1. Relational Database (right now MySQL or MariaDB, more will be supported soon)
2. An SMTP server for notifications

### Testing MariaDB

If you need a simple database implementation for testing, this [MariaDB deployment can be used](https://raw.githubusercontent.com/OpenUnison/kubeconeu/main/src/main/yaml/mariadb_k8s.yaml).  If you use this database, use the following configuration information:

| Configuration Option | Value |
| ---- | -------------------- |
| Host | `mariadb.mariadb.svc` |
| Port  | `3306` |
| User Name | `unison` |
| Password | `startt123` |
| Database Name | `unison` |

### Testing SMTP Server

If you don't have an SMTP server available, you can use the SMTP Blackhole we created to have a place to send email
without forwarding it to any recipients:

```
kubectl create ns blackhole
kubectl create deployment blackhole --image=tremolosecurity/smtp-blackhole -n blackhole
kubectl expose deployment/blackhole --type=ClusterIP --port 1025 --target-port=1025 -n blackhole
```

If you use the blackhole smtp service, use the following configuration information:

| Configuration Option | Value |
| -------------------- | ----- |
| Host | `blackhole.blackhole.svc` |
| Port | `1025` |
| Username | none |
| Password | none |
| TLS | `false` |

## Deployment

Once you have your [Authentication Portal](../deployauth/), database, and SMTP server deployed the next step is to
 add your database password and SMTP password to your `orchestra-secrets-source` `Secret` in the openunison namespace.  Assuming
you're using the testing MariaDB and SMTP Blackhole from above:

```
kubectl patch secret orchestra-secrets-source -n openunison --patch '{"data":{"OU_JDBC_PASSWORD":"c3RhcnR0MTIz","SMTP_PASSWORD":"ZG9lc25vdG1hdHRlcg=="}}'
```

Next, update your values.yaml by setting `openunison.enable_provisioning: true`, adding the below `az_rules`, and uncommenting the `database` and `smtp` sections of your values.yaml.
As an example, the below will work with the testing database and SMTP server:

```
openunison:
  enable_provisioning: true
  use_standard_jit_workflow: false
  az_groups:
  - k8s-cluster-k8s-administrators
  - k8s-namespace-administrators-k8s-*
  - k8s-namespace-viewer-k8s-*

database:
  hibernate_dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  quartz_dialect: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
  driver: com.mysql.jdbc.Driver
  url: jdbc:mysql://mariadb.mariadb.svc:3306/unison
  user: unison
  validation: SELECT 1

smtp:
  host: blackhole.blackhole.svc.cluster.local
  port: 1025
  user: "none"
  from: donotreply@domain.com
  tls: false
```

Next, update the orcehstra chart:

```
helm upgrade orchestra tremolo-betas/orchestra --namespace openunison -f /path/to/values.yaml
```

Wait for openunison-orchestra to finish creating and for your old container deployment.

***This will take much longer then the orchestra login portal alone.  The orchestra portal needs to wait for ActiveMQ to be available.***

Once done, update the `orchestra-login-portal` chart:

```
helm upgrade orchestra-login-portal tremolo-betas/orchestra-login-portal --namespace openunison -f /path/to/values.yaml
```

Next, login to your portal.  You'll see two "badges":

![NaaS Portal Login](../assets/images/ou-login-naas.png)

*ActiveMQ Console* - How you can view messages in queues, move messages from the dead letter queue.

*Operator's Console* - Search for users and apply workflows directly

You see these badges because the first user to login is provisioned as an administrator.  The dashboard and tokens are not there because you haven't yet been authorized for access to the cluster.  Your second, third, fourth, user, etc that logs in will not see any badges.

There are multiple ways for you to provision namespaces.  Find the option you want to determine how to provision namespaces and gain access to them.

## NaaS Models

OpenUnison provides three models out-of-the-box for managing and provisioning namespaces:

* **Internal Groups with Self Service** - This model uses groups managed by OpenUnison to provide access to your namespaces.  Use this model if you want to provide a self service model for accessing namespaces.  Using local management, once a namespace is created a user can "request access" to it and the owner of the namespace can approve the access.  There's no need for your cluster management staff to get involved.  You can also enable existing namespaces to have this functionality by adding an anotation.
* **External Groups** - Using external groups, you specify which groups from your identity provider manage access to your namespaces on creation.  This is useful when you want to drive access management from a central location.  This will work with any of the authentication methods supported by OpenUnison.  For Active Directory and Okta, you're able to select which groups to use wrather then having to type the names.
* **Hybrid Management** - You can enable both internal and external group management at the same time.  This is useful when you want to drive most authorization decisions via centralized groups from your identity provider but want the flexibility to explicitly enable access when needed.

You don't need to settle on one model initially.  You can start for instnce with external groups and later add internal groups with self service.  Next, we'll cover how to deploy each model.

### Internal Groups with Self Service

The internal groups with self service model will create a `Namespace` and groups inside of OpenUnison's database for namespace administrators, namespace viewers, and
namespace approvers.  There's no connection to your enterprise directory store and everything is self contained.  Users get access to namespace roles by
requesting access through the portal.  Namespace approvers can approve, or deny, the access.  New namespaces are requested from inside of the portal,
with openunison administrators being able to approve the namespace's creation.

To deploy the local management self service model, first enable internal groups by adding the following to your values.yaml:

```
openunison:
  .
  .
  .
  az_groups:
  - k8s-cluster-k8s-administrators
  - k8s-cluster-k8s-administrators-internal
  - k8s-namespace-administrators-k8s-*
  - k8s-namespace-viewer-k8s-*
  naas:
    groups:
      internal:
        enabled: false
```

Next, deploy the helm chart:

```
helm install cluster-management tremolo-betas/openunison-k8s-cluster-management -n openunison -f /path/to/values.yaml
```

Once deployed, login to OpenUnison.  The first user to login will be granted OpenUnison administrator and cluster administrator privileges.  

### External Groups

This model lets you use groups from your central authentication store to control who has access to namespaces.  When a namespace is requested and approved, `RoleBinding` objects are created that map to your central authentication store.  When using LDAP, Active Directory, or Okta you're able to pick groups.  When using OpenID Connect, SAML2, or GitHub you need to type in the names of the groups.  

Unlike the ***Internal Groups with Self Service*** NaaS, this mode does not use workflows to provide access to individual namespaces.  All namespace access is governed by your centralized groups.

To deploy the authentication groups model, first identify groups in your identity provider that will manage who are OpenUnison administrators (who will be able to approve the creation of new `Namespaces`) and another group to manage cluster management.  Then add the following to your values.yaml:

```
openunison:
  .
  .
  .
  use_standard_jit_workflow: false
  az_groups:
  - k8s-cluster-k8s-administrators
  - k8s-cluster-k8s-administrators-internal
  - k8s-namespace-administrators-k8s-*
  - k8s-namespace-viewer-k8s-*
  naas:
    groups:
      external:
        enabled: true
        adminGroup: k8s-admins
        clusterAdminGroup: k8s-admins
```

If you're using Active Directory, make sure to use the value for the group's `distinguishedName` attribute. 


As an example for Active Directory:

```
openunison:
  .
  .
  .
  use_standard_jit_workflow: false
  az_groups:
  - k8s-cluster-k8s-administrators
  - k8s-cluster-k8s-administrators-internal
  - k8s-namespace-administrators-k8s-*
  - k8s-namespace-viewer-k8s-*
  naas:
    groups:
      external:
        enabled: true
        adminGroup: "CN=openunison-admins,CN=Users,DC=ent2k12,DC=domain,DC=com"
        clusterAdminGroup: "CN=k8s_login_ckuster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com"
```


Finally, deploy the helm chart:

```
helm install cluster-management tremolo-betas/openunison-k8s-cluster-management -n openunison -f /path/to/values.yaml
```

#### Choosing Okta Groups

If you're using Okta as your identity provider and using the External Group Management model, you can tell OpenUnison to lookup groups instead of having to type them in when requesting a new `Namespace`.  To enable this feature, you'll need a token that can read groups from your Okta account.  

Once you have that token, add it to your `orchestra-secrets-source` `Secret` in the `openunison` `Namespace` using the 
key `OKTA_TOKEN`:

```
kubectl patch secret orchestra-secrets-source -n openunison --patch '{"data":{"OKTA_TOKEN":"c3RhcnR0MTIz"}}'
```

Next, in the `oidc` section of your values, add `type: okta`:

```
oidc:
  client_id: XXXX_YYYY
  issuer: https://XXXX.okta.com/
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
  type: okta
```

Finally, upgrade your chart deployment:

```
helm upgrade cluster-management tremolo-betas/openunison-k8s-cluster-management -n openunison -f /path/to/values.yaml
```

When you attempt to create a new `Namespace` you'll be presented with a list of up to ten groups from your Okta deployment.  As you type the first letters of the group you want the list will update.  You can click the name of the group you want to use.

#### Limiting AD/LDAP Groups

If you want to limit which groups can be chosen for managing access while using either Active Directory or LDAP, add `active_directory.group_search_base` to your values.yaml with the distinguished name of where you want groups to be searched for ***without your value of `active_directory.base`***.  For instnace if I want to limit groups to `cn=AWS,cn=users,dc=ent2k12,dc=domain,dc=com`, and my `active_directory.base`
is `cn=users,dc=ent2k12,dc=domain,dc=com`, the value for `active_directory.group_search_base` would be `cn=AWS`.  If you have already deployed 
`orchestra-k8s-cluster-management-by-group`, upgrade it with your new values.yaml:

```
helm upgrade cluster-management tremolo-betas/openunison-k8s-cluster-management -n openunison -f /path/to/values.yaml
```

When you attempt to create a new `Namespace` you'll be presented with a list of up to ten groups from your Active Directory or LDAP deployment.  As you type the first letters of the group you want the list will update.  You can click the name of the group you want to use.

### Hybrid Management

You can run both models at the same time.  This is useful when you want to use centralized management for the majority of access, but still use local management and self-service for edge cases.  Simply follow the steps for both models!