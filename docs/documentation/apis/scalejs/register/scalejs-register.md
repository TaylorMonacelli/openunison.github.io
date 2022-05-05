# ScaleJS Register

The Register API provides an API for building input to call a workflow, generally from a web application form.  Similar to `CallWorkflow`, this API adds additional functionality:

1. Generates metadata used to create a form or input, including data types and allowed values for drop down lists.
2. Performs data validation on input based on server side configuration.
3. Runs as the authenticated user, limiting potential security issues.

The [swagger definition](../../assets/yaml/swagger/api-scalejsregister.yaml) can be used to generate an API stub.

!!swagger api-scalejsregister.json!!