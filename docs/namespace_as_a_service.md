# Namespace as a Service

## Introduction

The OpenUnison Namespace as a Service (NaaS) portal gives your users control of their own corner of their cluster without involving cluster operations in the creation or access management for their `Namespace`.  What makes OpenUnison unique is that:

1. You can write objects either directly to the API server or to a git repository to support a GitOps workflow to `Namespace` creation
2. The NaaS portal can be applied to existing clusters to get immediate self service access
3. Collect additional metadata
3. Approvals can be customized and automated, for instance instead of always requiring an approval for a new `Namespace` you can have it automatically approved in certain situations
4. Support for "Day 2" operations such a resizing quotas and providing access to namespaces without running the kubectl command
5. Provisioning of non-Kubernetes resources, such as pipelines, git repos, etc.

<iframe width="560" height="315" src="https://www.youtube.com/embed/wKjFIs4ny48" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

There are three modes for using the OpenUnison NaaS to authorize access to a cluster:

1. OpenUnison Managed Groups - All groups are stored by OpenUnison with access managed internally by OpenUnison.  This gives you the most flexability in how your users access your cluster without any kind of outside dependencies.
2. Externally Managed Groups - When a user requests a `Namespace` be created, they specify groups from their identity provider.  Whenver a user logs in to use the cluster they're able to access namespaces based on their external identity.
3. Hybrid - When a `Namespace` is requested, both internally managed groups and external groups are used.

Of the three potential approaches, the Hybrid approach is the most popular.  This gives you the ability to use existing groups for majority of access while still letting OpenUnison support exceptions that always seem to be required.  The next section provides the details as to what the `Namespace` creation process entails and what gets created.

## Namespace Creation Process

From a systems' perspective, the default `Namespace` creation process will create:

| Object | Location | Description |
| ------ | ---------| ------------ |
| `Namespace` | Kubernetes or Git | The requested Namespace |
| Admin `RoleBinding`(s) | Kubernetes or Git |  `RoleBinding` for the `admin` `ClusterRole`.  One `RoleBinding` is created for each of external and internal groups depending on which authorization model is used. |
| View `RoleBinding`(s) | Kubernetes or Git |  `RoleBinding` for the `view` `ClusterRole`. One `RoleBinding` is created for each of external and internal groups depending on which authorization model is used. |
| Approval Group | Database | This group is used to determine who can approve access to the Admin and View roles.  The requester of the `Namespace` is added as the first user. |
| Namespace Admin Group(s) | Database | A group for internal and/or internal is created in the database to manage access to the Admin role for the `Namespace` |
| Namespace View Group(s) | Database | A group for internal and/or internal is created in the database to manage access to the View role for the `Namespace` |

If using git to provision objects, in addition to the above being provisioned into Git, the following objects are created:

| Object | Location | Description |
| ------ | ---------| ------------ |
| Git `Secret` | Kubernetes | An SSH private key used by OpenUnison to interact with the git repository that is responsible for supporting the new `Namespace` |

This provides the most basic `Namespace` capabilities.  You can customize this workflow to add `ResourceQuota` objects, additional roles, or additional metadata you may use to track charge-back.

## Deploying The NaaS Portal

### Before You Deploy

Before moving forward, you will need two additional components are required:

1. Relational Database - MySQL, MariaDB, PostgreSQL, and MS SQL Server are all supported
2. An SMTP server for notifications

#### Testing MariaDB

If you need a simple database implementation for testing, this [MariaDB deployment can be used](https://raw.githubusercontent.com/OpenUnison/kubeconeu/main/src/main/yaml/mariadb_k8s.yaml).  If you use this database, use the following configuration information:

| Configuration Option | Value |
| ---- | -------------------- |
| Host | `mariadb.mariadb.svc` |
| Port  | `3306` |
| User Name | `unison` |
| Password | `startt123` |
| Database Name | `unison` |

#### Testing SMTP Server

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

### Deployment

Once you have your [Authentication Portal](../deployauth/), database, and SMTP server deployed the next step is to
 add your database password and SMTP password to your `orchestra-secrets-source` `Secret` in the openunison namespace.  Assuming
you're using the testing MariaDB and SMTP Blackhole from above:



Next, update your values.yaml by setting `openunison.enable_provisioning: true`, `openunison.use_standard_jit_workflow: false`,  and uncommenting the `database` and `smtp` sections of your values.yaml.
As an example, the below will work with the testing database and SMTP server:

```
openunison:
  enable_provisioning: true
  use_standard_jit_workflow: false


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

The above is for MySQL or MariaDB.  For other databases:

#### Databases

##### MariaDB & MySQL

```
database:
  hibernate_dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  quartz_dialect: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
  driver: com.mysql.jdbc.Driver
  url: jdbc:mysql://mariadb.mariadb.svc:3306/unison
  user: unison
  validation: SELECT 1
```

##### PostgreSQL

```
database:
  hibernate_dialect: org.hibernate.dialect.PostgreSQLDialect
  quartz_dialect: org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
  driver: org.postgresql.Driver
  url: jdbc:postgresql://postgresql-db.postgres.svc.cluster.local:5432/unison
  user: postgres
  validation: SELECT 1
```

##### SQL Server

```
database:
  hibernate_dialect: org.hibernate.dialect.SQLServer2008Dialect
  quartz_dialect: org.quartz.impl.jdbcjobstore.MSSQLDelegate
  driver: com.microsoft.sqlserver.jdbc.SQLServerDriver
  url: jdbc:sqlserver://192.168.2.102:1433;databaseName=unison
  user: unison
  validation: SELECT 1
```

With your configuration updated, the next step is to choose which management model to use.  That is covered in the next section.


### NaaS Models

OpenUnison provides three models out-of-the-box for managing and provisioning namespaces:

* **Internal Groups with Self Service** - This model uses groups managed by OpenUnison to provide access to your namespaces.  Use this model if you want to provide a self service model for accessing namespaces.  Using local management, once a namespace is created a user can "request access" to it and the owner of the namespace can approve the access.  There's no need for your cluster management staff to get involved.  You can also enable existing namespaces to have this functionality by adding an anotation.
* **External Groups** - Using external groups, you specify which groups from your identity provider manage access to your namespaces on creation.  This is useful when you want to drive access management from a central location.  This will work with any of the authentication methods supported by OpenUnison.  For Active Directory and Okta, you're able to select which groups to use wrather then having to type the names.
* **Hybrid Management** - You can enable both internal and external group management at the same time.  This is useful when you want to drive most authorization decisions via centralized groups from your identity provider but want the flexibility to explicitly enable access when needed.

You don't need to settle on one model initially.  You can start for instnce with external groups and later add internal groups with self service.  Next, we'll cover how to deploy each model.

#### Internal Groups with Self Service

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
  naas:
    groups:
      internal:
        enabled: true
```

If you want to enable Hybrid Management, move on to the next section.  Otherwise skip straight to [deployment](#deployment).

#### External Groups

This model lets you use groups from your central authentication store to control who has access to namespaces.  When a namespace is requested and approved, `RoleBinding` objects are created that map to your central authentication store.  When using LDAP, Active Directory, or Okta you're able to pick groups.  When using OpenID Connect, SAML2, or GitHub you need to type in the names of the groups.  

Unlike the ***Internal Groups with Self Service*** NaaS, this mode does not use workflows to provide access to individual namespaces.  All namespace access is governed by your centralized groups.

To deploy the authentication groups model, first identify groups in your identity provider that will manage who are OpenUnison administrators (who will be able to approve the creation of new `Namespaces`) and another group to manage cluster management.  Then add the following to your values.yaml:

```
openunison:
  .
  .
  .
  use_standard_jit_workflow: false
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
  naas:
    groups:
      external:
        enabled: true
        adminGroup: "CN=openunison-admins,CN=Users,DC=ent2k12,DC=domain,DC=com"
        clusterAdminGroup: "CN=k8s_login_ckuster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com"
```


Next, determine if you can pre-load groups from an external identity provider, otherwise skip straight to [Deployment](#deployment).



##### Choosing Okta Groups

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

When you attempt to create a new `Namespace` you'll be presented with a list of up to ten groups from your Okta deployment.  As you type the first letters of the group you want the list will update.  You can click the name of the group you want to use.

##### Limiting AD/LDAP Groups

If you want to limit which groups can be chosen for managing access while using either Active Directory or LDAP, add `active_directory.group_search_base` to your values.yaml with the distinguished name of where you want groups to be searched for ***without your value of `active_directory.base`***.  For instnace if I want to limit groups to `cn=AWS,cn=users,dc=ent2k12,dc=domain,dc=com`, and my `active_directory.base`
is `cn=users,dc=ent2k12,dc=domain,dc=com`, the value for `active_directory.group_search_base` would be `cn=AWS`.  If you have already deployed 
`orchestra-k8s-cluster-management-by-group`, upgrade it with your new values.yaml:



When you attempt to create a new `Namespace` you'll be presented with a list of up to ten groups from your Active Directory or LDAP deployment.  As you type the first letters of the group you want the list will update.  You can click the name of the group you want to use.

##### GitHub Teams

If you're using GitHub for authentication, you can leverage Github teams to manage access to your namespaces.  OpenUnison can load your teams directly from GitHub making for a better UX when requesting new namespaces.  The first step is to [setup a GitHub App and download the private key](../applications/github/#registering-a-github-app).  Next create a `Secret` with that key in the `openunison` namespace.  Finally, in the `github` section of your values.yaml, set the orgnaization name, application id, and the name of the `Secret` with the key that has the private key generated by GitHub:

```yaml
github:
  client_id: abcdefghij123
  teams: TremoloSecurity/
  naas:
    appid: "123456"
    org: "TremoloSecurity"
    secret:
      name: githubapp
      key: github.pem
```

Once you deploy OpenUnison, you'll be able to choose teams from a list, and type them in to find a specific one.

#### Hybrid Management

You can run both models at the same time.  This is useful when you want to use centralized management for the majority of access, but still use local management and self-service for edge cases.  Simply follow the steps for both models!


### Deployment

The `ouctl` command you used when deploying the authentication portal will detect the changes to your configuration and update the deployment accordingly.  Once you have your secret files and updated yaml, run:

```
ouctl install-auth-portal -b /path/to/db/secret -t /path/to/smtp/secret /path/to/openunison-values.yaml
```

This run will take longer then the authentication portal deployment takes because it's also deploying an ActiveMQ service to manage workflow tasks.  Once completed, you can move on to your first login.

### Your First Login

Next, login to your portal.  You'll see two "badges":

![NaaS Portal Login](../assets/images/ou-login-naas.png)

*ActiveMQ Console* - How you can view messages in queues, move messages from the dead letter queue.

*Operator's Console* - Search for users and apply workflows directly

You see these badges because the first user to login is provisioned as an administrator.  The dashboard and tokens are not there because you haven't yet been authorized for access to the cluster.  Your second, third, fourth, user, etc that logs in will not see any badges.

## Next Steps

With the Namespace as a Portal deployed, you can go to the [user's manual](/documentation/naas/) to see how users can login to begin requesting new namespaces and managing access!

## Alternate Deployment Steps

### Manual Deployment

First, update the `orchestra-secrets-source` `Secret` with the passwords for your database and SMTP service:

```
kubectl patch secret orchestra-secrets-source -n openunison --patch '{"data":{"OU_JDBC_PASSWORD":"c3RhcnR0MTIz","SMTP_PASSWORD":"ZG9lc25vdG1hdHRlcg=="}}'
```

Next, update the orcehstra chart:

```
helm upgrade orchestra tremolo/orchestra --namespace openunison -f /path/to/values.yaml
```

Wait for openunison-orchestra to finish creating and for your old container deployment.

***This will take much longer then the orchestra login portal alone.  The orchestra portal needs to wait for ActiveMQ to be available.***

Once done, update the `orchestra-login-portal` chart:

```
helm upgrade orchestra-login-portal tremolo/orchestra-login-portal --namespace openunison -f /path/to/values.yaml
```

Finally, install the `cluster-management` chart:

```
helm install cluster-management tremolo/openunison-k8s-cluster-management -n openunison -f /path/to/values.yaml
```

Once the `cluster-management` chart is deplotyed, you can continue to **Your First Login**