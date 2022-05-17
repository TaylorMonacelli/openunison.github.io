# Namespace as a Service Portal

This section details the steps used to interact with the Namespace as a Service Portal.  In this section you will learn how to gain access as an administrator, request a new namespace be approved and how to gain access to a namespace that has been provisioned.

## Cluster Authentication and Access

The [same instructions for accessing your clusters](/documentation/login-portal/) works in the NaaS portal.

## Administrative Access

There are two sets of administrator roles:

1. **administrators** - Whether external or internal, an **administrator** is responsible for approving assignment of cluster administrator and for the creation of new namespaces.
2. **cluster-administrators** - Whether external or internal, a **cluster-administrator** has the Kubernetes `cluster-admin` `ClusterRoleBinding`.

If you're using the external management model, these roles are assigned automatically based on your groups in your identity provider and there's no furth action needed.  If you're using the internal groups management or hybrid management models there are two ways a user can gain access to these roles.  Either via self service or by being assigned by an administrator.

### Administrator Self Service Access

If a user wishes to become an adminitrator, first the user must login to the NaaS self service portal:

![Self Service Request Administrator](/assets/images/manual/naas/access-administrator.png)

1. Click on **Request Access**
2. Choose **OpenUnison Internal Groups** so that it is now in bold
3. Next to `administrators-internal`, click **Add To Cart**
4. Click on **Check Out** in the menu bar

At this point the request is in your user's cart and can be checked out:

![Checkout Request](/assets/images/manual/naas/checkout-administrator.png)

1. Provide a reason for why the access to administrator is needed.
2. Click **Submit Request** to complete the request.

A current administrator will need to approve this request.  You can see who is able to approve the request by **Viewing Reports**, namely the **My Open Requests** report.

### Cluster Administrator Access

Cluster administrators have the Kubernetes `cluster-admin` `ClusterRoleBinding`.  They have the ability to do anything inside of the cluster, but can not approve the creation of new namespaces requested by other users.  To request cluster administration access, login to the NaaS Portal:

![Self Service Request Cluster Administrator](/assets/images/manual/naas/access-clusteradmin.png)

1. Click on **Request Access**
2. Choose **Kubernetes Administration** so that it is now in bold
3. Next to `Local Deployment Cluster Administrator`, click **Add To Cart**
4. Click on **Check Out** in the menu bar

At this point the request is in your user's cart and can be checked out:

![Checkout Request](/assets/images/manual/naas/checkout-clusteradmin.png)

1. Provide a reason for why the access to administrator is needed.
2. Click **Submit Request** to complete the request.

A current administrator will need to approve this request.  You can see who is able to approve the request by **Viewing Reports**, namely the **My Open Requests** report.

## Requesting a New Namespace

Any user can request to create a new namespace.  Once requested, It's up to the OpenUnison administrators to approve the request.  In order to request a new namespace, first login to the NaaS portal.

![New Namespace](/assets/images/manual/naas/new-ns-badge.png)

Click on the **New Kubernetes Namespace** badge.  The next screen will depend on which management model you choose.  

![New Namespace Form](/assets/images/manual/naas/new-ns-scren.png)

| Field | Description | Mode |
| ----- | ----------- | ---- |
| Cluster | The cluster where you want to create the new namespace | Internal, External, Hybrid |
| Namespace Name | A unique name, must comply with Kubernetes naming standards | Internal, External, Hybrid |
| Administrator Group | If External or Hybrid mode is enabled, the name of the group from the identity provider that will have the `admin` role for this namespace | External, Hybrid |
| Viwer Group | If External or Hybrid mode is enabled, the name of the group from the identity provider that will have the `view` role for this namespace | External, Hybrid |
| Internal Access Groups | In Hybrid mode, determins if the `tremolo.io/request-access: enabled` label should be placed on the namespace to allow users to request access.  | Hybrid |
| Reason | Why the namespace needs to be created | Internal, External, Hybrid |

If using external or hybrid mode and your remote identity provider is supported (currently Okta and Active Directory), you'll be able to choose your groups.  Otherwise you'll need to know the name of the group ahead of time and enter it in the text box.

Once you click **Submit Request**, a current administrator will need to approve this request.  You can see who is able to approve the request by **Viewing Reports**, namely the **My Open Requests** report.  Once approved and your namespace has been created, you'll be notified via an email.  After that, users can login to request access to your namespace or you can add them to your groups in the identity provider so they can start working in their namespaces.

## Requesting Access to a Namespace

In Hybrid or an Internal management modes, users can login to request access to any namespace with the `tremolo.io/request-access: enabled` label.  First, login to the NaaS portal and:

![Request Namespace Access](/assets/images/manual/naas/request-ns.png)

1. Click on **Request Access** in the menu bar
2. For your cluster, click on the arrow next to it to show the available options
3. Choose eithert **Administrators** or **Viewers** depending on what's being requested
4. Next to the namespace you want access to, click on **Add To Cart**
5.  In the menu bar, click on **Check Out** to complete your request

On the checkout screen:

![Checkout Request](/assets/images/manual/naas/request-ns-checkout.png)

1. Provide a reason for why the access to administrator is needed.
2. Click **Submit Request** to complete the request.

A namespace owner will need to approve this request.  You can see who is able to approve the request by **Viewing Reports**, namely the **My Open Requests** report.

You'll receive an email once your access is approved.  On your next login, you'll be able to access the approved namespace as part of your request. 

## Access Approvals

Once a request is made, approvers will receive email notifications that there is an open request.  Requests are approved in the NaaS portal.  When you have an open request and login to the portal, there will be a small red number next to **Open Approvals** in the menu bar, as seen below:

![Portal with Approvals](/assets/images/manual/naas/approvals-portal.png)

Click on **Open Approvals** and the list of approvals that are assigned to you are shown:

![Open Approvals](/assets/images/manual/naas/approvals-review.png)

Click on **Review** on an approval, and the review screen will appear:

![Review Approval](/assets/images/manual/naas/approval-review.png)

This screen provides information about the request and the current state of the user making the request:

1. Provide your justification for approving or denying the request
2. Choose either approving or denying the request.  If you choose to deny the request, and the user already has access to groups in question, their access will be removed.

Once you choose to approve or deny the requst, the next screen is similar but for confirmation.  There's nothing to edit.  You can either confirm your choices or go back.

![Confirm Approval](/assets/images/manual/naas/approval-confirm.png)

This step will:

Depending on your apprval status:
  
  * **If approved** - Notify the user of your decision and let them know they can begin using the resource they requested
  * **If denied** - Notify the user why the request was denied

Finally, any other user that could be responsible for this approval will have it removed from their list of open approvals.

## Reports

OpenUnison has several reports that you can use to find the status of open requests, view who approved requests, and track which users have accessed your clusters.  Clicking on the **Reports** link in the upper right hand corner of the menu bar will load the reports.  Below are the reports that come pre-built with the NaaS portal:

| Report | Description | Accessible By |
| ------ | ----------- | ------------- |
| My Open Requests | Lists all open requests and who can approve them. | Everyone |
| Approvals Completed By Me | Lists all approvals completed by the logged in user | Everyone |
| Open Approvals | List of all open requests and who can approve them | OpenUnison Administrators |
| Completed Approvals | List of all approvals between a set of dates and who completed the approval | OpenUnison Administrators |
| Single User Change Log | List of all attributes and objects that have been created that are associated with the user provided when running the report | OpenUnison Administrators |
Change Log for Period | List of all attributes and objects that have been created or changed between the two dates provided | OpenUnison Administrators |
| Dormant Users | List of users that haven't logged in for 30 days | OpenUnison Administrators |