# Certificate Management

## How do I change OpenUnison's certificates?

OpenUnison's certificate when deployed in Kubernetes is hosted by the Ingress controller, not by the OpenUnison container its self.  When used for the login portal, we want to supply the CA certificate for two reasons:

1. So it can be embedded in the `kubectl` command correctly
2. So that the dashboard SSO works properly when validating the login process

Before moving forward you'll need:

1. A certificate with subject alternative names for your portal (`OU_HOST` / `network.openunison_host`) and your dashboard (`K8S_DASHBOARD_HOST` / `network.dashboard_host`).  If using impersonation, the impersonation host is needed too (`K8S_API_HOST` / `network.api_server_host`)
2. The certificate authority (CA) certificate that signed your certificate from #1
3. Any intermediate certs needed to complete the chain

Once you have your certificates and keys:

1. Delete the `ou-tls-certificate` secret in the `openunison` namespace - `kubectl delete secret ou-tls-certificate -n openunison`
2. Recreate the `ou-tls-certificate` secret - `kubectl create secret tls ou-tls-certificate --cert=/path/to/chain.pem --key=/path/to/key.pem -n openunison` - **NOTE** `chain.pem` should be your entire certificate chain, including the CA and all intermediate certs.
3. Edit the `orchestra` `openunison` object - `kubectl edit openunison orchestra -n openunison`.  Look for `tls_secret_name: ou-tls-certificate` and delete the entire array entry its a part of:
```
      - create_data:
          ca_cert: true
          key_size: 2048
          server_name: k8sou.apps.192-168-159-168.nip.io
          sign_by_k8s_ca: false
          subject_alternative_names:
          - k8sdb.apps.192-168-159-168.nip.io
        import_into_ks: certificate
        name: unison-ca
        tls_secret_name: ou-tls-certificate
```
Continue editing the CR, find the section called `trusted_certificates` (you may already have an entry).  Add an an item called `unison-ca` to the list with the value of `pem_data` being the base64 encoded PEM for your CA's certificate.  For example:

```
    trusted_certificates:
    - name: ldaps
      pem_data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURORENDQWh5Z0F3SUJBZ0lRYlJOajZSS3F0cVZQdlc2NXFaeFhYakFOQmdrcWhraUc5dzBCQVFVRkFEQWkKTVNBd0hnWURWUVFEREJkQlJFWlRMa1ZPVkRKTE1USXVSRTlOUVVsT0xrTlBUVEFlRncweE5EQXpNamd3TVRBMQpNek5hRncweU5EQXpNalV3TVRBMU16TmFNQ0l4SURBZUJnTlZCQU1NRjBGRVJsTXVSVTVVTWtzeE1pNUVUMDFCClNVNHVRMDlOTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUEyczlKa2VOQUhPa1EKMVFZSmdqZWZVd2Nhb2dFTWNhVy9rb0ErYnU5eGJyNHJIeS8yZ04va2M4T2tvUHV3Si9uTmxPSU8rcytNYm5YUwpMOW1VVEM0T0s3dHJrRWppS1hCK0QrVlNZeTZpbVhoNnpwQnROYmVaeXgrcmRCbmFPdjNCeVpSbm5FQjhMbWhNCnZIQSs0Zi90OWZ4LzJ2dDZ3UHgvL1ZnSXE5eXVZWVVRUkxtMVdqeVVCRnJaZUdvU3BQbTBLZXdtK0IwYmhtTWIKZHlDKzNmaGFLQytVazFOUG9kRTI5NzNqTEJaSmVsWnhzWlk0MFd3OHpZUXdkR1lJYlhxb1RjKzFhL3g0ZjFFbgptNEFOcWdnSHR3K05xOHpoc3MzeVR0WStVWUtEUkJJTGRMVlpRaEhKRXhlMGtBZWlzZ014SS9iQndPMUhickZWCit6U25rK252Z1FJREFRQUJvMll3WkRBekJnTlZIU1VFTERBcUJnZ3JCZ0VGQlFjREFRWUlLd1lCQlFVSEF3SUcKQ2lzR0FRUUJnamNVQWdJR0NDc0dBUVVGQndNRE1CMEdBMVVkRGdRV0JCVHlKVWZZNjZ6WWJtOWkweGVZSHVGSQo0TU43dURBT0JnTlZIUThCQWY4RUJBTUNCU0F3RFFZSktvWklodmNOQVFFRkJRQURnZ0VCQU01a3o5T0tOU3VYCjh3NE5PZ25mSUZkYXpkMG5QbElVYnZEVmZRb055OVEwUzFTRlVWTWVrSVBOaVZoZkd6eWE5SXdSdEdiMVZhQlEKQVEyT1JJekhyOEEycjVVTkx4M21GanBKbWVPeFF3bFYwWCtnOHMrMjUzS1ZGeE9wUkU2eXlhZ24vQnh4cHRUTAphMVo0cWVRSkxENDJsZDFxR2xSd0Z0VlJtVkZaelZYVnJwdTdOdUZkM3Zsbm5PL3FLV1hVK3VNc2ZYdHNsMTNuCmVjMWt3MUV3cTJqbks4V0ltS1RRNy85V2JhSVkwZ3g4bW93Q0pTT3NScTBURTd6Sy9ONTVkck4xd1hKVnhXZTUKNE4zMmVDcW90WHk5ajlsemRrTmE3YXdiOXEzOG5XVnhQK3ZhNWpxTklEbGxqQjZ0RXh5NW4zczd0NktLNmc1agpUWmdWcXJaMyttcz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    - name: unison-ca
      pem_data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNvRENDQVlnQ0NRRGhlNzN5cDQwQjNEQU5CZ2txaGtpRzl3MEJBUXNGQURBU01SQXdEZ1lEVlFRRERBZHIKZFdKbExXTmhNQjRYRFRJd01ETXlPVEUzTXpJeE1Wb1hEVE13TURNeU56RTNNekl4TVZvd0VqRVFNQTRHQTFVRQpBd3dIYTNWaVpTMWpZVENDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLRFpSNmRqCm56czB2ZGhPdzFRdGIwWm9rb2xxTUJwYlpkcGMrY3EyRWFxRThCWEtTZDJMOGFSQldPVVdBNFhVMDRPclEyUjIKUU83b0NybERBR3dMRmpORktXZlBDNG5PMFhYcmVydEN2UEdrNGFKWndmSDArSThnTmw4TjhwL09QQklQOVFaaQp4K0RYekJibGFkRlZydGVtVTh5L1JqMGZXb1pJY0NPdzk5bDBhTGtxb21jSGo0UFBtYTZqYWYrOFZxSnZDT0ZMCktBUEVoeVEzYWFIYlZGNFV3TGlDMWNjenhFWWxXcnNIOUFleVpzdGovdG5FcEl3MzlVQkNMUVZLaHFFZVl5a0YKU3VGekNTcFJZZmNoUFBtK3NMQjY3cmoyc01KUFhyU1pDZHdjR0NSeW5GUDNjcnE3bkpRdndmM2tZV21vTDc5bQo3T2ZsamlweTJzeC81RThDQXdFQUFUQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFnWFlsZmNpNjhHMmFMbkhqCjFjcTZmNmgvZGdEbWxXeUpGZG9qYnNBRUFCb0plUHhxY1dWOGZsVGZTTkRYY25hWmtQOEdhQXo2T2NpbXEwYlgKVWdEUnVIWlM5ZmZBQXJrK29DZTVxZVhNN3IvQUJrdWVZTDBiSVN5elowQjBBUnBtVldRd2MrS1E3TFltYWQxVQpiVVU3Qi9seHdIcDYrcXA1RVFrQ1hWeWRNanc5TS9pU3VoTXhYZjdkRGZ4R0hJSHgwZU9kVjhFQ0cyc29xTUw4ClZPU1E4OEZ1YlZQVWVBOHhTWURIWVYwaWZVTW4wOFUyY1FmeHd2Tlp0Y3FhTDlmUWZQTFExYU9NWDV2S0lUdkwKS29QczJUUkhKek44WTJENVNxdThTckRDY25DQXN4RFNMUXR6Q2cxelY1V3VCWXM3UHdnWExyZExBcVdUb3UvbAptN0VJMVE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
```

Save and close your editor so the CR can update.  Make sure to update your API servers to use the new certificate.

## Using a Commercially Signed Ingress Certificate

OpenUnison deploy's a self-signed certificate for your Ingress to make it easy to start using.  Most production deployments do not use this certificate but instead use a wildcard or commercially signed certificate.  To stop using the self signed certificate:

1. Delete the `ou-tls-certificate` secret in the `openunison` namespace - `kubectl delete secret ou-tls-certificate -n openunison`
2. Edit the `orchestra` `openunison` object - `kubectl edit openunison orchestra -n openunison`.  Look for `tls_secret_name: ou-tls-certificate` and delete the entire array entry its a part of:
```
      - create_data:
          ca_cert: true
          key_size: 2048
          server_name: k8sou.apps.192-168-159-168.nip.io
          sign_by_k8s_ca: false
          subject_alternative_names:
          - k8sdb.apps.192-168-159-168.nip.io
        import_into_ks: certificate
        name: unison-ca
        tls_secret_name: ou-tls-certificate
```
Save and close your editor so the CR can update.  Make sure to update your API server to remove the `--oidc-ca-file` flag if not using impersonation.  Once the OpenUnison pods are redeployed there should no longer see the OpenUnison certificate when you login to the token screen.  Also, the oulogin plugin will work now as well.