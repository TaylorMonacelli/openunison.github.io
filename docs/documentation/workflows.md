# Workflows

## Introduction

OpenUnison can run a series of tasks, or a workflow, to provision access or infrastructure.  There are multiple use cases for using workflows in OpenUnison:

* **Just-In-Time Provisioning** - When providing SSO to an application that stores its own identity data, OpenUnison can synchronize this data from your identity provider as the user logs in.
* **Granting Access to Resources** - If a namespace or cluster has been provisioned, and you wish to grant someone access.
* **Bootstrapping Infrastrcture** - A user needs a new namespace to be created, OpenUnison can create it, and the additional objects across databases, git repositories, and controllers to make namespace useable.

When using OpenUnison, there are two kinds of workflows:

* **Synchronous** - Each task of the workflow is run in the same thread and immediately, if there's a failure then the entire workflow fails.
* **Asynchronous** - Each task of the workflow is places in a message queue and executed asynchronously.  If a task fails, then it is send to a dead letter queue for later processing.

When using OpenUnison as a login portal, only synchronous workflows can be run.  When using the Namespace as a Service portal, either synchronous or asynchronous workflows may be run.  This document will provide the form of workflows and instructions on how to build them.

## Workflow YAML

Workflows in OpenUnison are stored as `Workflow` objects in your Kubernetes API server.  A basic workflow will look like:

```yaml linenums="1"
---
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: my-workflow
  namespace: openunison
spec:
  description: This is my workflow
  inList: false
  label: This is a great workflow
  orgId: 687da09f-8ec1-48ac-b035-f2f182b9bd1e
  dynamicConfiguration:
    dynamic: false
    className: ""
    params: []
  tasks: |-
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
      params:
        message: in my workflow
```

The above yaml stores the metadata for the workflow as well as the individual tasks.  Tasks are embeded yaml, and are **NOT** part of the `Workflow` CRD.  To make it easier to debug workflows, OpenUnison has an admission controller that will report back errors when trying to submit a workflow.  Below are the components of the workflow configuration:

| Line Number | Description |
| ----------- | ------------|
| 30 | A detailed description of the workflow, may include request parameters by using `$parameter$` |
| 31 | When using Namespace as a Service, should this workflow be listed in the UI and requested by users? |
| 32 | A shorter label that will be displated in the NaaS portal. May include request parameters by using `$parameter$` |
| 33 | The `Org` that the workflow is a member of.  If `inList` is `true`, then the logged in user must be authorized per the `Org` to see and request this workflow from the NaaS portal |
| 34 - 37 | `dynamicConfiguration` allows for a single workflow to be created to handle multiple subjects.  For instance, this is how the NaaS portal provides a workflow for access to each namespace |
| 39 - 42 | The workflow tasks for this workflow as an embeded YAML array.  If there is a problem with the embeded YAML, the admission controller will report back the error |

With the basic form of the `Workflow` defined, the next step is define what tasks are available for workflows.

## Workflow Tasks

OpenUnison defines a handful of workflow tasks, with the rest being filled in by "custom tasks", which are written in Java to support more complex logic.  You don't need to write tasks in Java though, as OpenUnison supports JavaScript for tasks too.

### provision

This task is used to push user data to a provisioning target in an idempotent way.  This task should be safe to run with the same inputs multiple times with only differences updated. This task does NOT support sub tasks.

```yaml linenums="1"
- taskType: provision
  sync: true
  target: jitdb
  setPassword: false
  onlyPassedInAttributes: false
  attributes:
  - uid
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | If `false`, the `provision` task will add attributes and groups in a non-destructive way.  For instance if the `sn` attribute is not included in the provision task, it will not be removed.  The same is true of groups.  It `true`, the `provision` task will make the target object match the current workflow representation. Groups that aren't in the workflow are removed |
| 3 | The OpenUnison `Target` to run the task against |
| 4 | If `true`, the task attempts to set the user's password, usually `false` |
| 5 | If `true`, the task will only attempt to synchronize attributes that are in the workflow's user object.  Usually `false`.  It's better to explicitly set attributes via the `attributes` option. |
| 6 - 7 | List of attributes the task should work with. |

### ifNotUserExists

This task will execute sub tasks if-and-only-if there is not a user in the internal virtual directory that matches the value of the attribute specified in the current user's context.

```yaml linenums="1"
- taskType: ifNotUserExists
  target: jitdb
  uidAttribute: uid
  onSuccess:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: user DOES NOT exist
  onFailure:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: user DOES exist
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The OpenUnison `Target` to run the task against |
| 3 | The name of the user attribute to use for lookup |
| 4 - 8 | The list of tasks to run if the user does not exist |
| 9 - 13 | The list of tasks to run if the user does exist |

### addAttribute

Adds an attribute to the user’s object or to the workflow’s request object.

```yaml linenums="1"
- taskType: addAttribute
  name: cluster
  value: k8s-$cluster$
  remove: false
  addToRequest: true
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the attribute, may include request parameters by using `$parameter$`|
| 3 | The value to add or remove from the attribute, may include request parameters by using `$parameter$` |
| 4 | If `true`, the attribute value is removed.  If `false`, the attribute value is added |
| 5 | If `true`, the attribute is added to the workflow's request object.  If `false`, the attribute is added to the workflow's user object |


### addGroup

Adds the named group to the user. NOTE - The group name is NOT a distinguished name, its a name that will be mapped to a group in a provisioning target. This usually means mapping to the cn attribute.

```yaml linenums="1"
- taskType: addGroup
  name: $openunison_grouptoremove$
  remove: true
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the group, may include request parameters by using `$parameter$`|
| 3 | If `true`, the group is removed from the user object.  If `false`, the group is added |


### approval

The approval step provides for a user to act on a request. and provide either their consent or to disapprove of the request. Each approval can have a set of authorization rules to determine who can make the approval. In addition to authorizations escalation rules can be defined that let the authorization rules change. Finally, if all escalations are exhausted a failure policy can be defined to allow for a request to be re-assigned or rejected.

```yaml linenums="1"
- taskType: approval
  emailTemplate: A new namespace has been requested
  mailAttr: mail
  failureEmailSubject: Namespace not approved
  failureEmailMsg: |-
    Because:
    ${reason}
  label: Create New Namespace - $cluster$ - $nameSpace$
  approvers:
  - scope: group
    constraint: cn=administrators,ou=groups,ou=shadow,o=Tremolo
  - scope: group
    constraint: cn=administrators,ou=groups,ou=shadow,o=Tremolo
  escalationPolicy:
    escalations:
    - executeAfterTime: 5
      validateEscalationClass: ""
      executeAfterUnits: day
      azRules:
      - scope: group
        constraint: cn=supervisors,ou=groups,ou=shadow,o=Tremolo
    failure:
      action: assign
      azRules:
      - scope: group
        constraint: cn=auto-fail,ou=groups,ou=shadow,o=Tremolo
  onSuccess:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: approval was successful!
  onFailure:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: approval failed!

```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | An email template to send to approvers, may include request parameters by using `$parameter$` |
| 3 | The name of the attribute on the workflow's user that has their email attribute |
| 4 | Subject of the email to send to the user if the request is denided, may include request parameters by using `$parameter$` |
| 5 - 7 | Body of the email to send to the user if the request is denied, may include request parameters by using `$parameter$` |
| 8 | The label for the approval, seen in the Orchestra portal.  may include request parameters by using `$parameter$` |
| 9 - 13 | Defines who is able to approve requests.  May include request parameters by using `$parameter$`. An authorization rule is made of a `scope` (one of `dn`,`group`,`filter`,`custom`) and a `constraint`, which defines what the rule is.  The combination of `dn` and `o=Tremolo` tells OpenUnison to authorize any user in the internal LDAP virtual directory.  If you wanted to only allow users in the group `cn=argocd-users,ou=Groups,DC=domain,DC=com` you would use `filter` as the `scope` and `(groups=cn=argocd-users,ou=Groups,DC=domain,DC=com)` as the `constraint` because OpenUnison internally represents groups in the Kubernetes integration as attributes instead of seperate objects. Multipls rules may be listed. |
| 14 - 26 | An optional policy can be set to escalate requests that aren't acted on in a specific amount of time. |
| 15 - 21 | Lists each escalation rule in order |
| 16 - 21 | Specific escalation rule |
| 16 | The amount of time before the escalation rule triggers |
| 17 | Class that is called when verifying an escalation should execute, must implement `com.tremolosecurity.proxy.az.VerifyEscalation` class |
| 18 | The unit of time (sec,min,hour,day,month,year) |
| 19 - 21 | Defines who is able to approve requests.  May include request parameters by using `$parameter$`. An authorization rule is made of a `scope` (one of `dn`,`group`,`filter`,`custom`) and a `constraint`, which defines what the rule is.  The combination of `dn` and `o=Tremolo` tells OpenUnison to authorize any user in the internal LDAP virtual directory.  If you wanted to only allow users in the group `cn=argocd-users,ou=Groups,DC=domain,DC=com` you would use `filter` as the `scope` and `(groups=cn=argocd-users,ou=Groups,DC=domain,DC=com)` as the `constraint` because OpenUnison internally represents groups in the Kubernetes integration as attributes instead of seperate objects. Multipls rules may be listed. |
| 22 - 26 | Defines what happens if the last escalation is not acted on in its `escalationAftertime`, this usually is used to assign to an automatic failure user so the request can be closed. |
| 23 | Either "leave" to do nothing, or "assign" to assign the request to the user(s) defined in the azRules |
| 24 - 26 | Defines who is able to approve requests.  May include request parameters by using `$parameter$`. An authorization rule is made of a `scope` (one of `dn`,`group`,`filter`,`custom`) and a `constraint`, which defines what the rule is.  The combination of `dn` and `o=Tremolo` tells OpenUnison to authorize any user in the internal LDAP virtual directory.  If you wanted to only allow users in the group `cn=argocd-users,ou=Groups,DC=domain,DC=com` you would use `filter` as the `scope` and `(groups=cn=argocd-users,ou=Groups,DC=domain,DC=com)` as the `constraint` because OpenUnison internally represents groups in the Kubernetes integration as attributes instead of seperate objects. Multipls rules may be listed. |
| 27 | Tasks to execute if the approval succeeds |
| 32 | Tasks to execute if the request is denied |

### customTask

This task provides a mechanism by which custom logic may be added to a workflow. See individual custom tasks for parameters that support parameters.

```yaml linenums="1"
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
  params:
    message: I'm here!!!
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the class that implements the com.tremolosecurity.provisioning.util.CustomTask interface |
| 3 | Parameters for the task |

There are numerous OpenUnison custom tasks.  Look for them based on their application.  If you want to build your own custom task, [the easiest way is using JavaScript](../../customization/custom_provisioning_tasks/).

### delete

The delete task will delete a user in an individual provisioning target.

```yaml linenums="1"
- taskType: delete
  target: k8s
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the provisioning target to delete the user from |


### ifAttrExists

This task will run if a user object in the workflow has a particuler attribute, regardless of the value.

```yaml linenums="1"
- taskType: ifAttrExists
  name: githubTeams
  onSuccess:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: githubTeams exists
  onFailure:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: githubTeams does not exist
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the attribute to look for, may include request parameters by using `$parameter$` |
| 3 - 7 | Tasks to run if the attribute exists |
| 8 - 12 | Tasks to run if the attribute does not exist |


### mapping

The mapping task will create a copy of the user object based on the mapping rules. Any changes made inside of the mapping to the user object will NOT affect the user object in the workflow outside of the mapping. Unlike other tasks that have sub-tasks, mappings can only have "onSuccess" sub tasks. Anything in "onFailure" will be ignored.

```yaml linenums="1"
- taskType: mapping
  strict: true
  map:
    - targetAttributeName: uid
      targetAttributeSource: uid
      sourceType: user
  onSuccess:
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
    params:
      message: inside of the mapping
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | If true, the mapped user object will ONLY contain attributes explicitly named in the mapping.  If false, then any attribute not mentioned will be added to the user object with their present value. |
| 3 - 6 | The mapping |
| 4 | The name of the attribute in the new user object, supports parameters.  If `TREMOLO_USER_ID` is used, the mapped user's ID will be set |
| 5 | The value to be used for the new attribute, supports parameters.  If `TREMOLO_USER_ID` is used, the value is pulled from the user object's identity |
| 6 | Describes what the targetAttributeSource is.  Possible values are:<br/>`static` - Takes the value from targetAttributeSource as is<br/>`user` - Takes the existing value from the attribute named in targetAttributeSource<br/>`composite` - Use a composite of multiple attributes and static values by containing attribute names in `${attributeName}`<br/>`custom` - targetAttributeSource is the name of a class that implements com.tremolosecurity.provisioning.mapping.CustomMapping |
| 7 - 11 | The tasks to run within the `mapping` context |

### notifyUser

This task provides a way to send an email to the requester of the workflow. This can be used to notify the user of a successful execution, request more information, etc. Emails are sent from the server specified in the approvals section of the configuration. If the requester is different then the subject of the workflow, then the requester is notified UNLESS the unison.sendToSubject property is stored in the workflow’s request object. The value does not matter.

```yaml linenums="1"
- taskType: notifyUser
  subject: Namespace has been created
  mailAttrib: mail
  msg: |-
    Your namespace has been created!
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The subject of the email notification, supports parameters |
| 3 | The name of the attribute that stores the user's email address |
| 4 | The message to send.  Attributes may be added between `${}`, supports parameters |

### resync

When executing a just-in-time provisioning workflow, for instance when using identity federation, once the user's object is created in downstream targets the user?s object in Unison will need to be ?refreshed?. This task updates the internal OpenUnison object.

```yaml linenums="1"
- taskType: resync
  keepExternalAttrs: false
  changeRoot: true
  newRoot: o=Tremolo
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | If true, the user object will maintain any attributes that were not loaded from the internal virtual directory. |
| 3 | If true, OpenUnison will reload the user's object from a different search base then the one that was originally used to find the user |
| 4 | If changeRoot is checked, the root of the virtual directory to use to find the user; supports parameters |

### callWorkflow

This task allows for another workflow to be called. This allows for the creation of modular workflows. For instance a modular workflow can be created that requires 2 approvals before provisioning to a resource. This workflow can be included in a self-service request from the portal and a helpdesk application with the same results without having to duplicate the workflow.

```yaml linenums="1"
- taskType: callWorkflow
  name: some-other-workflow
```

| Line Number | Description |
| ----------- | ------------|
| 1 | The task type |
| 2 | The name of the workflow to call |