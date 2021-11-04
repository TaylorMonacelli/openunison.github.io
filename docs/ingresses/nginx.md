# NGINX Integration

OpenUnison works out-of-the-box with the NGINX Ingress Controller.  Specify `network.ingress_type=nginx` in your values. yaml.  The
helm chart will generate the proper `Ingress` object for you, along with sticky sessions.  You can add annotations to the 
`network.ingress_annotations` object in your values.yaml for additional annotations. 

[Return to deployment](../../deployauth#pre-requisites)