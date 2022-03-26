# AzureAD

OpenUnison provides a unique benefit when working with AzureAD.  In addition to providing secure authentication to both the kubectl CLI tool, the Kubernetes Dashboard, and other cluster applications, OpenUnison can also translate AzureAD group ids into their display names.

When working with OpenID Connect and AzureAD, you can include group information in the `id_token`, but it goes to your application as an id, not as a name.  For instance, if you looked at the `id_token` for a simple OIDC integration with AzureAD, it would look like:

```json
{
  "aud": "e3edb579-187a-49b6-bf80-5418b818bbb9",
  "iss": "https://login.microsoftonline.com/61cbe426-d3ca-4ebd-8ca4-e47e354a85bb/v2.0",
  "iat": 1648235554,
  "nbf": 1648235554,
  "exp": 1648235854,
  "auth_time": 1648235852,
  "family_name": "Admin",
  "given_name": "K8s",
  "name": "K8s Admin",
  "oid": "6c2e1de2-b864-4e1b-a01f-0baa4da67c06",
  "preferred_username": "k8sadmin@marcboorshteintremolosecuri.onmicrosoft.com",
  "rh": "0.ARMAJuTLYcrTvU6MpOR-NUqFu3m17eN6GLZJv4BUGLgYu7kTAHU.",
  "roles": [
    "212702a6-f1e0-4b7e-aa12-f66a610b119c"
  ],
  "sub": "HGha_KjmLta679qnxCbZmiNrDj7IwcAxlbh5o1M3Kz4",
  "tid": "61cbe426-d3ca-4ebd-8ca4-e47e354a85bb",
  "uti": "oMNX2g3CKkquh4VI5yXxAA",
  "ver": "2.0"
}
```

Notice that the `roles` claim has an entry of `212702a6-f1e0-4b7e-aa12-f66a610b119c`.  This is the id of the administrator's group in the example AzureAD domain, `k8s-administrators`.  While you could reference this group id in your RBAC `RoleBinding` and `ClusterRoleBinding`, this could become difficult to manage quickly.  

OpenUnison solves this problem by looking up the user in AzureAD for you and translating the group ids into the group's display name, making it much easier to define your RBAC bindings.  

Before configuring OpenUnison, first create an application in your AzureAD domain.

## Configuring AzureAD

The first step is to define a new **App registration** in your AzureAD configuration.  Use a name that is descriptive. For the **Redirect URI** use **Web** and specify the URL for OpenUnison's redirect uri.  It will be `https://OU_HOST/auth/oidc` where `OU_HOST` is the value from `network.openunison_host` in your values.yaml.

![AzureAD Application Registration](../../../assets/images/identity-providers/azuread/1.png)

Once your application is registered, the next step is to create a client secret.  Click on **Add a certificate or secret** in the upper left hand section of the screen.

![Create Client Secret](../../../assets/images/identity-providers/azuread/2.png)

Click on **+ New client secret**, give it a description and a life span.  How long the secret is valid is based on your requirements and compliance needs.  Once you click **Add**, the secret will become available for you to copy.  Copy it, and store it to the `OIDC_CLIENT_ID` key in the `orchestra-secrets-source` `Secret`.  ***If using `echo` to based64 encode the data, make sure to use `echo -n` to avoid adding an extra newline!***

With the client secret created and stored in your cluster, the next step is to setup your token.  Next, click on **Token configuration** on the left hand side.  Next, click on **Add optional claim** and choose `email`, `family_name`,`given_name`, and `preferred_username` and click **Add**.  When AzureAD asks if you want to add the scopes automatically, agree.  Adding these claims isn't required, but will make it much easier to manage your integration.

![Add optional claims](../../../assets/images/identity-providers/azuread/3.png)

The last configuration step is to enable the correct APIs and provide organizational consent.  Click on **API permissions** on the left hand side.  The `email`, `profile`, and `User.Read` delegated permissions are already present.  Click on **+ Add a permission**, Choose **Microsoft Graph**, and then **Application Permissions**.  Under **User** check `User.Read.All`.  Under `Group` check `Group.Read.All`.  Under **GroupMember** check `GroupMember.Read.All`.  Click **Add permission**.

![API Permissions](../../../assets/images/identity-providers/azuread/4.png)

Next, click on **Grant admin consent for Default Directory** and answer **Yes**.  This will allow OpenUnison to lookup the user's groups.  

![Grant Organization Consent](../../../assets/images/identity-providers/azuread/4.png)

Return to the **Overview** section to configure OpenUnison.

## Configuring OpenUnison

To configure OpenUnison, uncomment the `oidc` section of your values.yaml and update as shown:

```yaml
oidc:
  # get your client_id from your app registration's Application (client) ID
  client_id: e3edb579-187a-49b6-bf80-5418b818bbb9
  # replace TENNANT_ID with the value from Directory (tenant) ID from the overview screen of your app registration
  issuer: https://login.microsoftonline.com/TENNANT_ID/v2.0/
  user_in_idtoken: true
  domain: ""
  scopes: openid email profile
  claims:
    sub: preferred_username
    email: email
    given_name: given_name
    family_name: family_name
    display_name: name
    groups: roles
```

In addition to setting `oidc.client_id` and `oidc.issuer`, change :

1. `oidc.user_in_idtoken` to `false`
2. Remove `groups` from `oidc.scopes`
3. Change `oidc.claims.sub` to `preferred_username`
4. Change `oidc.claims.groups` to `roles`

With the updates to the `oidc` section, add 

```yaml
azure:
  # replace with the value from Directory (tenant) ID from the overview screen of your app registration
  tennant_id: 61cbe426-d3ca-4ebd-8ca4-e47e354a85bb
```

The final configuration change is to set `openunison.include_auth_chain` to `azuread-load-groups`.  ***NOTE: `openunison.include_auth_chain` is NOT included in the default values yaml.*** This last step will tell OpenUnison to run our custom authentication mapping.  The next section covers the details as to what is happening in this custom mapping.

All configuration is complete.  The next step is to install the `orchestra-login-azuread` chart:

```
helm install orchestra-login-azuread tremolo/orchestra-login-azuread -n openunison -f /path/to/openunison-values.yaml
```

With this chart installed, you may complete the [Deploy the Portal](../../deployauth#deploy-the-portal) step.  The details for how the `orchestra-login-azuread` chart maps from group ids to names is in the next section.

## Mapping Technical Details

The details of how the mapping occurs invovles creating two new objects in your cluster:

1. AzreAD provisioning target - Targets are how OpenUnison communicates with remote services.  The AzureAD target knows how to authenticate to AzureAD and load users based on their `userPrinicipalName` and translates the group ids the user is a member of into group names.
2. Custom authentication chain - A custom authentication chain, written in JavaScript, retrieves the authenticated user, loads the AzureAD target, finds the user, and stores the user's groups in the authenticated user's object.

The target is straight forward.  It uses the same identity as OpenUnison's OIDC configuration.  The additional permissions we added gives it the ability to prerform the look.

The custom authentication is where the actual mapping happens.  Here's the javascript snippet:

```javascript linenums="1"
function doAuth(request,response,as) {
    // setup classes that we can use from Java
    Attribute = Java.type("com.tremolosecurity.saml.Attribute");
    ProxyConstants = Java.type("com.tremolosecurity.proxy.util.ProxyConstants");
    GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");
    HashMap = Java.type("java.util.HashMap");
    System = Java.type("java.lang.System");
    
    // get the session data needed
    var session = request.getSession();
    var holder = request.getAttribute(ProxyConstants.AUTOIDM_CFG);

    var ac = request.getSession().getAttribute(ProxyConstants.AUTH_CTL);

    // Load the user from AzureAD by their UserPrincipalName (user@domain)
    var upn = ac.getAuthInfo().getAttribs().get("uid").getValues().get(0);
    var azuread = GlobalEntries.getGlobalEntries().getConfigManager().getProvisioningEngine().getTarget("azuread");
    var userFromAzureAD = azuread.findUser(upn,new HashMap());

    // remove the old groups, then add the ones from the lookup to AzureAD
    var memberof = ac.getAuthInfo().getAttribs().get("memberOf");
    if (memberof == null) {
        memberof = new Attribute("memberOf");
        ac.getAuthInfo().getAttribs().put("memberOf",memberof);
    }
    memberof.getValues().clear();
    memberof.getValues().addAll(userFromAzureAD.getGroups());

    as.setExecuted(true);
    as.setSuccess(true);
    holder.getConfig().getAuthManager().nextAuth(request, response,session,false);

}
```

| Line Numbers | Description |
| ------------ | ----------- |
| 1 | This function must be defined to execute an authentication step |
| 2 - 7 | Makes classes available from Java to the JavaScript engine |
| 10 - 13 | Load the user from previous authentication mechanisms in the chain |
| 15 - 18 | Loads the user from AzureAD using the provisioning target |
| 21 - 27 | Add the groups to the user data for the authentication process |
| 29 - 31 | Tell OpenUnison this step in the chain was successful and to continue |