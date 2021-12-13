# Customized Just-In-Time Login Workflow

It's common for deployments to need custom actions after a user logs in.  These actions might mean onboarding into specific groups,
adding objects to your cluster, or any other number of actions that you require.  A common example is creating a namespace specifically
for the logged in user in which they are the `admin` and are able to use as a sandbox.  Assuming you are using the OpenUnison 
[Namespace as a Service](../../namespace_as_a_service/) you can use this [workflow](https://gist.github.com/mlbiam/a7d8e6cb5ee4afb49f2bb19a0da8a726) 
as a starting point.  This workflow:

1. Converts the user's login id to something that will work as a `Namespace`.  For instance, if the user's login is an email address, the `@` and `.` are converted to numeric codes.  
2. Prepends `user-ns-` to the user id to determine the new `Namespace` name.
3. Creates a group for managing access to the new `Namespace`.
4. Adds the user to the new group.
5. Creates a `Namespace`.
6. Creates a `RoleBinding` for the `ClusterRole` `admin` to the newly created group
7. Reloads the user's identity to provide access to the new `Namespace`

Assuming you don't want to make any changes, to deploy:

```
kubectl apply -f https://gist.githubusercontent.com/mlbiam/a7d8e6cb5ee4afb49f2bb19a0da8a726/raw/c49a129ca6f1962866f9972d3854960b634b5538/create-user-namespace.yaml
```

Next, update your values.yaml to include `openunison.post_jit_workflow` with the name of your custom `Workflow`, in this case `create-user-namespace` and update your `orchestra-login-portal` helm deployment:

```
helm upgrade orchestra-login-portal tremolo/orchestra-login-portal --namespace openunison -f /path/to/values.yaml
```

The next time a user logs into OpenUnison, they'll have their own `Namespace`!  This workflow can be customized in any number of ways, such as by adding
a `ResourceQuota` in the `Workflow` to limit how much resources the user's sandbox can consume.  