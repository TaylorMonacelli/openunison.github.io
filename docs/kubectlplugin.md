# Kubectl Plugin

This plugin will launch a browser to log you into your Kubernetes cluster from the command line.  This plugin requires OpenUnison to be integrated with Kubernetes.  One of the Orchestra portals will likely fit your use case (https://github.com/openunison/).  This plugin:

1. Launches a browser to authenticate you to OpenUnison (and your identity provider)
2. Creates a context and user in your kubectl configuration
3. Sets the new configuration as your default context

![Video of CLI Login](assets/images/login-cli.gif)

There is no pre-configuration that needs to happen.  OpenUnison provides all the configuration for your cluster, just as if logging into OpenUnison and getting the configuration from the token screen and pasting it into your cli window.

## Installation

The simplest way to install this plugin is via the krew plugin manager:

```
$ kubectl krew install oulogin
```

## Running

The plugin takes one parameter, `host`, the host of your OpenUnison.  There is no need to have an existing kubectl configuration file.  If one exists, the cluster configuration will be added to it.

```
$ kubectl oulogin --host=k8sou.apps.domain.com
```

## FAQ

### What is the difference between this plugin and the oidc-login plugin?

The `oidc-login` plugin is a generic plugin that will work with any OpenID Connect identity provider.  It requires that you pre-configure kubectl for use with the OpenID Connect identity provider.  The `oulogin` plugin is designed to work with OpenUnison and creates your kubectl configuration for you.  There's nothing to pre-configure on the client.

### The login process complains about not trusting a certificate, can I use an untrusted cert?

The OpenUnison certificate **MUST** be trusted by your client.  The OpenUnison certificate can be obtained by logging into OpenUnison and clicking on the token screen.

### Can I Use oulogin With Multiple Clusters?

Yes.  If you run `kubectl oulogin --host=cluster1` it will set your cluster configuration for cluster1.  If you then run `kubectl oulogin --host=cluster2` if will set your configuration for cluster2.  Running `kubectl oulogin --host=cluster1` again will just switch your context.

### Will The Plugin Prompt Me To Login, Even With a Valid Session?

No, if your `id_token` is still OK and your still able to interact with the API server then you aren't prompted to login again.
