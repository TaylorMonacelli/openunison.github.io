# Login Portal

The Login Portal for Kubernetes provides three functions out-of-the-box:

1. `kubectl` authentication/sso without having to build and distribute a configuration file
2. Secure access to the (Kubernetes Dashboard)[https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/]
3. A `kubectl` plugin that supports zero configuration access

You can configure your portal to support other applications too.

## `kubectl` Access

Using the OpenUnison portal you can obtain a `kubectl` command that will create your configuration file for you.  This includes configuring certificates as well.  You can get both a Windows and a *nix command.  There's no plugin needed for this approach, as it uses the out-of-the-box `kubectl` OpenID Connect integration.

First, login to the portal.  You'll be presented with "badges" that provide links to authorized resources.  Click on the badge that says **Kubernetes Tokens**:

![Kubernetes Tokens](../../../assets/images/manual/login/tokens.png)

### Linux/Unix Login

If you are using a *nix platform, such as Linux or MacOS, click on the **Copy** icon next to **kubectl Command**:

![Kubernetes Tokens - *nix](../../../assets/images/manual/login/token-copy-nix.png)

This will copy the `kubectl` command into your copy & paste buffer.  Finally, paste this command into your favorite terminal and hit *Enter*:

![Kubernetes Tokens - paste](../../../assets/images/manual/login/token-paste.png)

The above image shows an access error, but notice that it's showing the user logged in with OpenID Connect!

### Windows / Powershell Login

When using Powershell on Windows, click on the **Copy** icon next to **kubectl Windows Command**:

![Kubernates Tokens - Windows](../../../assets/images/manual/login/token-copy-windows.png)

This coppies the Powershell command into your copy & paste buffer.  Finally, paste into Powershell:

![Kubernates Tokens - Windows Paste](../../../assets/images/manual/login/token-paste-win.png)

The above image shows an access error, but notice that it's showing the user logged in with OpenID Connect!

### Logout

For both Powershell and Linux/Unix varients, your token will be refreshed by `kubectl` automatically.  If your session times out, or you click the **Logout** button on the portal, your session will be terminated and you'll need to login again to get a new `kubectl` configuration.

![Kubernetes Tokens - Logout](../../../assets/images/manual/login/logout.png)

Once you're logged out, within a few minutes any `kubectl` sessions you have will be terminated and you'll see a message like:

```
PS C:\Users\marcb> kubectl get nodes
Unable to connect to the server: failed to refresh token: oauth2: cannot fetch token: 401 Unauthorized
Response:
```

When you see this message, it means it's expired and you need to login again to get new tokens.

## Secure Dashboard Access

Accessing the Kubernetes Dashboard securely is a great way to make quick updates or view the status of your cluster when you don't want to setup an API session with `kubectl`.  First, login to OpenUnison and click on the **Kubernetes Dashboard** badge:

![Kubernetes Dashboard](../../../assets/images/manual/login/dashboard-badge.png)

This will open a new tab with the dashboard in it using your permissions.

## `kubectl` Plugin

If you want a CLI first experience, the `oulogin` plugin for `kubectl` will launch a browser for you and create your `kubectl` configuration without having to start in a browser.  

This plugin will launch a browser to log you into your Kubernetes cluster from the command line.  This plugin requires OpenUnison to be integrated with Kubernetes.  One of the Orchestra portals will likely fit your use case (https://github.com/openunison/).  This plugin:

1. Launches a browser to authenticate you to OpenUnison (and your identity provider)
2. Creates a context and user in your kubectl configuration
3. Sets the new configuration as your default context

![Video of CLI Login](../../../assets/images/manual/login/login-cli.gif)

There is no pre-configuration that needs to happen.  OpenUnison provides all the configuration for your cluster, just as if logging into OpenUnison and getting the configuration from the token screen and pasting it into your cli window.

### Installation

The simplest way to install this plugin is via the krew plugin manager:

```
$ kubectl krew install oulogin
```

### Running

The plugin takes one parameter, `host`, the host of your OpenUnison.  There is no need to have an existing kubectl configuration file.  If one exists, the cluster configuration will be added to it.

```
$ kubectl oulogin --host=k8sou.apps.domain.com
```

### FAQ

#### What is the difference between this plugin and the oidc-login plugin?

The `oidc-login` plugin is a generic plugin that will work with any OpenID Connect identity provider.  It requires that you pre-configure kubectl for use with the OpenID Connect identity provider.  The `oulogin` plugin is designed to work with OpenUnison and creates your kubectl configuration for you.  There's nothing to pre-configure on the client.

#### The login process complains about not trusting a certificate, can I use an untrusted cert?

The OpenUnison certificate **MUST** be trusted by your client.  The OpenUnison certificate can be obtained by logging into OpenUnison and clicking on the token screen.


## What Groups Am I a Member Of?

Once you have logged into OpenUnison, click on your username in the upper left hand corner to see what groups you are a member of:

![View My Groups](../../../assets/images/manual/login/view-groups.png)

This will open a screen that will tell you what your login id (`sub`) is and what groups you're a member of under the **Roles** heading:

![My Groups](../../../assets/images/manual/login/my-groups.png)

The groups listed here are what Kubernetes will be looking for in `ClusterRoleBinding` and `RoleBinding` objects.