# Certificate Management

## How do I change OpenUnison's certificates?

OpenUnison's certificate when deployed in Kubernetes is hosted by the Ingress controller, not by the OpenUnison container its self.  When used for the login portal, we want to supply the CA certificate for two reasons:

1. So it can be embedded in the `kubectl` command correctly
2. So that the dashboard SSO works properly when validating the login process

Before moving forward you'll need:

1. A certificate with subject alternative names for your portal (`network.openunison_host`) and your dashboard (`network.dashboard_host`).  If using impersonation, the impersonation host is needed too (`network.api_server_host`)
2. The certificate authority (CA) certificate that signed your certificate from #1
3. Any intermediate certs needed to complete the chain

Once you have your certificates and keys:

1. Delete the `ou-tls-certificate` secret in the `openunison` namespace - `kubectl delete secret ou-tls-certificate -n openunison`
2. Recreate the `ou-tls-certificate` secret - `kubectl create secret tls ou-tls-certificate --cert=/path/to/chain.pem --key=/path/to/key.pem -n openunison` - **NOTE** `chain.pem` should be your entire certificate chain, including the CA and all intermediate certs.  You may also allow a tool like cert-manager to generate your certificate either directly or by specifying the correct annotations on your Ingress controller.
3. Update your `values.yaml` file to specify `network.createIngressCertificate=false`.
4. If your certificate isn't signed by a well known CA, such as Let's Encrypt, base64 encode the PEM certificate and add it to 
the `trusted_certs` section of your values.yaml with the name `unison-ca`:


```
trusted_certs:
  - name: unison-ca
    pem_b64: LS0tLS1CRUdJTiBDRVJUSU...
```

Finally, update both your `orchestra` and `orchestra-login-portal` helm deployments:

```
helm upgrade orchestra tremolo/orchestra --namespace openunison -f /path/to/values.yaml
helm install orchestra-login-portal tremolo/orchestra-login-portal --namespace openunison -f /path/to/values.yaml
```
