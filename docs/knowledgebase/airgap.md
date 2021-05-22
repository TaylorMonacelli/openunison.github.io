# Air-Gap Implementation

To deploy OpenUnison in an air-gap environment you need to import for the following images into your local repository.  The table notes the image, its function, the chart its referenced in and the Helm chart values to set to customize which image to use.

| Image | Function | Chart | Value Name |
| ----- | -------- | ----- | ---------- |
|docker.io/tremolosecurity/openunison-k8s-operator|OpenUnison Operator|tremolo/openunison-operator|`image`|
|docker.io/tremolosecurity/activemq-docker:latest|ActiveMQ Image Maintained by Tremolo Security|tremolo/openunison-*|`amq_image`|
|docker.io/tremolosecurity/kubernetes-artifact-deployment:1.1.0|Check certificates, update them as needed|tremolo/openunison-*|`cert_update_image`|
|docker.io/tremolosecurity/openunison-*|Orchestra Image|tremolo/openunison-*|`image`|
