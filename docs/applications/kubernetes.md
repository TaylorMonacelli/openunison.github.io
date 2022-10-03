# Kubernetes Integration

This page has common resources that can be used for integrating and managing Kubernetes clusters.  In addtion to providing SSO access to your clusters, OpenUnison can provision objects into your clusters.

## Workflow Tasks

### Wait For Status

When provisioning an object that can take some time and is asynchronous, it's helpful to be able to pause the workflow until a certain object is created and a status is set.  For instance, provisioning a vCluster via the ClusterAPI can take a few minutes.  Your workflow needs to wait until the vCluster is ready before provisioning OpenUnison into it.  This task pauses the workflow until an object has been created and a status has been met.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.WaitForStatus
  params:
    # control plane cluster
    holdingTarget: k8s
    # namespace that holds the target for the cluster to test
    namespace: openunison
    # target that points to the cluster you wish to test
    target: $cluster$
    # The URI of the object in the target cluster to test
    uri: /apis/apps/v1/namespaces/$nameSpace$/statefulsets/vcluster
    # Label for this test
    label: wait-for-vcluster
    # List of conditions that must be met in JSON Path notation
    conditions:
    - .status.readyReplicas=1
    - .status.replicas=1
```

In addition to adding this task, make sure the `oujob` **wait-for** has been created:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: OUJob
metadata:
  name: wait-for
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.jobs.WaitForJob
  cronSchedule:
    dayOfMonth: '*'
    dayOfWeek: '?'
    hours: '*'
    minutes: '*'
    month: '*'
    seconds: '*/10'
    year: '*'
  group: admin
  params:
    - name: target
      value: k8s
    - name: namespace
      value: openunison
```