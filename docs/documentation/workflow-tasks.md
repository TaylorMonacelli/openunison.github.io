# Workflow Tasks

## AsyncCallWorkflow

Launches another workflow asynchronously.  This will generate a new workflow in the audit database that will start and finish outside of the context of the original workflow.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.AsyncCallWorkflow
  params:
    # The name of the workflow to call
    workflowName: vcluster-post-namespace-create
    # The uid attribute of the user object passed into the workflow
    uidAttributeName: sub
    # The reason field for the workflow call
    workflowReason: "deploy vcluster for $nameSpace$"
```