# Monitoring OpenUnison with Prometheus

OpenUnison has a `/metrics` endpoint that exposes all of the built in Java metrics from the prometheus simple client as well as the number of open sessions.  This endpoint is secured as OpenUnison is part of security and you don't want to leak information about your security systems.  By default, the service account for the Kube Prometheus project (https://github.com/coreos/kube-prometheus) is authorized but that can be changed.

Update your values.yaml by adding `PROMETHEUS_SERVICE_ACCOUNT` to the `openunison.non_secret_data` object with the name of the `ServiceAccount`
you want to authorize access for:

```
openunison:
  replicas: 1
  non_secret_data:
    PROMETHEUS_SERVICE_ACCOUNT: system:serviceaccount:monitoring:prometheus-k8s
```

Next, create an RBAC role and binding to allow the monitoring stack to query the `openunison-orchestra` `Service`:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
    audit2rbac.liggitt.net/version: v0.7.0
  labels:
    audit2rbac.liggitt.net/generated: "true"
    audit2rbac.liggitt.net/user: system-serviceaccount-monitoring-prometheus-k8s
  name: monitoring-list-services
  namespace: openunison
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - pods
  - services
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    audit2rbac.liggitt.net/version: v0.7.0
  labels:
    audit2rbac.liggitt.net/generated: "true"
    audit2rbac.liggitt.net/user: system-serviceaccount-monitoring-prometheus-k8s
  name: monitoring-list-services
  namespace: openunison
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: monitoring-list-services
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
```

Finally, create a service monitor in the `monitoring` namespace:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: prometheus-operator
  name: orchestra
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 30s
    port: openunison-secure-orchestra
    scheme: https
    targetPort: 8443
    tlsConfig:
      insecureSkipVerify: true
  namespaceSelector:
    matchNames:
    - openunison
  selector:
    matchLabels:
      app: openunison-orchestra
```

Wait a few minutes, and you should see something similar to this in your openunison logs:
```
[2020-04-02 10:21:51,121][XNIO-1 task-10] INFO  AccessLog - [AuSuccess] - metrics - https://10.244.0.20:8443/metrics - username=system:serviceaccount:monitoring:prometheus-k8s,ou=oauth2,o=Tremolo - 20 / oauth2k8s [10.244.0.14] - [fefe1c571fbe12338abf0d5ba2fe4283a4a6d0def]
[2020-04-02 10:21:51,121][XNIO-1 task-10] INFO  AccessLog - [AzSuccess] - metrics - https://10.244.0.20:8443/metrics - username=system:serviceaccount:monitoring:prometheus-k8s,ou=oauth2,o=Tremolo -
           [10.244.0.14] - [fefe1c571fbe12338abf0d5ba2fe4283a4a6d0def]
```
You'll also see the metrics available in your Prometheus dashboard.