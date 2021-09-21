# Custom Provisioning Tasks

The OpenUnison workflow engine provides several built in tasks as well as pre-built tasks.  If you need custom logic, you can create
custom tasks in Java or JavaScript.  Creating custom tasks in JavaScript is an easy way to embed custom logic in your workflows.  
See the example custom task:

```
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.JavaScriptTask
  params:
    uidAttribute: uid
    javaScript: |-
      // if there's any pre-initialization code, you can put it here

      // the init function is called when the workflow is loaded
      // it is meant for loading configuration options.
      // task - com.tremolosecurity.provisioning.core.WorkflowTask
      // params - Map<String, Attribute>
      function init(task,params) {
          // the state map is used to store information you want
          // available for each run, like configuration attributes
          // everything stored in this dictionary MUST be serializable

          state.put("uidAttribute",
                    params.get("uidAttribute").getValues().get(0));
      }


      // reInit is called every time a task is deserialized and is ready to be called
      // this is a good time to rebuild ephemeral objects like connections
      // This function should return true if you want to continue the workflow,
      // false if you want the workflow to stop and be marked as completed
      // task - com.tremolosecurity.provisioning.core.WorkflowTask
      function reInit(task) {

      }

      // doTask is where you will do your work.  
      // user - com.tremolosecurity.provisioning.core.User
      // request - Map<String, Object>
      function doTask(user,request) {
          var OpenShiftTarget = Java.type("com.tremolosecurity.unison.openshiftv3.OpenShiftTarget");
          var sub = OpenShiftTarget.sub2uid(user.getAttribs().get(state.get("uidAttribute).get(0)));
          var namespaceName = "dev-user-ns-" + sub;
          request.put("nameSpace",namespaceName);

          // return true to continue the workflow
          return true;
      }
  
```

This task creates a namespace based on the logged in user's name.  Since a user id can have characters that don't comform with 
Kubernetes' requirements, we convert the uid to something that can be stored as the `metadata.name` attribute of the `Namespace`.
To do this, we need to know which attribute stores the user id.  We get this from the configuration of our custom task in the `params`
section of our configuration.  We can get this configuration in the `init` function.  OpenUnison workflows are serialized and stored
in state, so any data that you need between function calls should be stored in the `state` Map.  

The `doTask` function is called on execution where most of the work is done.  OpenUnison is built on Java and all classes available
to a Java custom task are available to a custom task written in JavaScript.  See [GraalVM's Java Interoperability](https://www.graalvm.org/reference-manual/js/JavaInteroperability/) documentation for details on how to interact with Java.
