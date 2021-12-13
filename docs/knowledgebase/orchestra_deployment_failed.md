#Failed Orchestra Deployment

If the result of deploying the orchestra chart looks like:

```
helm install orchestra tremolo/orchestra --namespace openunison -f Book-Second-Edition-Work/chapter5/openunison-values.yaml 
Error: failed pre-install: pod test-orchestra-orchestra failed
```

The pre-configuration tests failed.  These tests go through many of the most common misconfiguration issues with OpenUnison.  It's
not exhaustive, but will catch the most likely problems in your configuration.  The first step is to inspect the logs of the `test-orchestra-orchestra` pod:

```
kubectl logs test-orchestra-orchestra  -n openunison
Checking all three hosts are unique
network.openunison_host : k8sou.192-168-2-114.nip.io
network.dashboard_host : k8sou.192-168-2-114.nip.io
network.api_server_host : k8sapi.192-168-2-114.nip.io
network.openunison_host, network.dashboard_host, and network.api_server_host MUST be different from each other
```

In this case, my `network.openunison_host` and `network.dashboard_host` are the same.  These need to be unique, so I need to fix that in my values.yaml.
Once I have fixed it, to re-run:

```
helm delete orchestra -n openunison
helm install orchestra tremolo/orchestra --namespace openunison -f /path/to/values.yaml
NAME: orchestra
LAST DEPLOYED: Fri Jul  9 17:48:25 2021
NAMESPACE: openunison
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Now you can continue with your deployment.

## Error Messages and How To Fix

**network.openunison_host, network.dashboard_host, and network.api_server_host MUST be different from each other**

The deployment failed because your three `network` host names were not unique.  Each of `network.openunison_host`, `network.dashboard_host`, 
and `network.api_server_host` ***MUST*** be unique in order for OpenUnison to function properly.  For details as to why, see [Host Names and Networking](../../deployauth/#host-names-and-networking).

**network.XYZ is not a host name**

All of `network.openunison_host`, `network.dashboard_host`, and `network.api_server_host` ***MUST*** be fully qualified DNS domain names.  Do not
specify a URL or a port.  These hosts are used to create `Ingress` rules.  The setting named in the log will tell you which host name to fix.

When using impersonation on managed clusters, it's a common misconfiguration to specify your API server's URL for `network.api_server_host`.  The `network.api_server_host` is the host name for your API server prody, not the URL of the actual API server.  OpenUnison will derive the correct URL for your API server from it's
own Pod.  You don't need to specify it.

**Impersonation enabled, but network.api_server_host not set**

When `enable_impersonation` is set to `true`, the `network.api_server_host` must be set to the host name you want to use for your api server proxy.
See [Host Names and Networking](../../deployauth/#host-names-and-networking) for details.

**Certificate XYZ not properly base64 encoded**

When adding a certificate to the list of `trusted_certs`, your PEM file ***MUST*** be base64 encoded.  For instance if you want to include a certificate:

```
base64 -w 0 <<EOF
-----BEGIN CERTIFICATE-----
MIIDezCCAmOgAwIBAgIEGj0QFzANBgkqhkiG9w0BAQsFADBtMQwwCgYDVQQGEwNk
ZXYxDDAKBgNVBAgTA2RldjEMMAoGA1UEBxMDZGV2MQwwCgYDVQQKEwNkZXYxDDAK
BgNVBAsTA2RldjElMCMGA1UEAxMcYXBhY2hlZHMuYWN0aXZlZGlyZWN0b3J5LnN2
YzAgFw0yMTA3MDUwMDUzMjhaGA8yMTIxMDYxMTAwNTMyOFowbTEMMAoGA1UEBhMD
ZGV2MQwwCgYDVQQIEwNkZXYxDDAKBgNVBAcTA2RldjEMMAoGA1UEChMDZGV2MQww
CgYDVQQLEwNkZXYxJTAjBgNVBAMTHGFwYWNoZWRzLmFjdGl2ZWRpcmVjdG9yeS5z
dmMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC6SUABQIFdbtpdbwXA
ONZnRUBQA0Er+hVbi1PsVJsZiIFf8ID/1epO7C8BUbUsuSwVZG9dBHWz4DPpYE++
vvlkLK5pGbNrPl7Ewn5rRKDNOyTyeCApS3ISlEmbiSPR0nawyrJh68PCB5mDR53f
hulXOCGSwvM7dp3dOsyEBdeVL7i1gnBMb/p7NXu3yZiVh9iKzjhCkfwj/El56ZPq
bk/8kP7LAu1odbVNFFILyrPzHPESr77Jkxw/++ONhmnP6PPbSqmFm+qEDahPjpEf
lqGZcpl8FtUpsLnI+pxnR9ye9eCiT5nh8eLHhnEE71iTMkolkHwqJnqWTwfQvoX9
C6HDAgMBAAGjITAfMB0GA1UdDgQWBBTc8iCSNy6xR3c99MZbFT883DK5WDANBgkq
hkiG9w0BAQsFAAOCAQEAN0I0frCK7X4dGFjF/1iow3QL/m70zeHziPUQEQGN5pE3
r2VY/FGXgMWKEIzGkhLenBQtqDNOnn//Rc+F6js5FCNUxZOexzIvhIacC8aHsEjE
TgrR99cjQWlw1aIQtaHdJejawXqNtaME1+xFRAA3BYAuOpoyopCCh1wIi1/BO8NT
VaZjw1AO1wVTi2wIKBIJugCvOGXdKwc0n/B8o4QBJRrIQdBgo2U61AnC1ic8oGwG
qzCuWGCzvcjoq5aMq9bKF24pPh4wqfLfvFuclbaHRPbJjqOytWx3s8MWIoECsvXK
xHbboJu4sP0/4A0d8+nN9r51O1jQZXwtoxpzqSWfeg==
-----END CERTIFICATE-----
EOF
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURlekNDQW1PZ0F3SUJBZ0lFR2owUUZ6QU5CZ2txaGtpRzl3MEJBUXNGQURCdE1Rd3dDZ1lEVlFRR0V3TmsKWlhZeEREQUtCZ05WQkFnVEEyUmxkakVNTUFvR0ExVUVCeE1EWkdWMk1Rd3dDZ1lEVlFRS0V3TmtaWFl4RERBSwpCZ05WQkFzVEEyUmxkakVsTUNNR0ExVUVBeE1jWVhCaFkyaGxaSE11WVdOMGFYWmxaR2x5WldOMGIzSjVMbk4yCll6QWdGdzB5TVRBM01EVXdNRFV6TWpoYUdBOHlNVEl4TURZeE1UQXdOVE15T0Zvd2JURU1NQW9HQTFVRUJoTUQKWkdWMk1Rd3dDZ1lEVlFRSUV3TmtaWFl4RERBS0JnTlZCQWNUQTJSbGRqRU1NQW9HQTFVRUNoTURaR1YyTVF3dwpDZ1lEVlFRTEV3TmtaWFl4SlRBakJnTlZCQU1USEdGd1lXTm9aV1J6TG1GamRHbDJaV1JwY21WamRHOXllUzV6CmRtTXdnZ0VpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFDNlNVQUJRSUZkYnRwZGJ3WEEKT05ablJVQlFBMEVyK2hWYmkxUHNWSnNaaUlGZjhJRC8xZXBPN0M4QlViVXN1U3dWWkc5ZEJIV3o0RFBwWUUrKwp2dmxrTEs1cEdiTnJQbDdFd241clJLRE5PeVR5ZUNBcFMzSVNsRW1iaVNQUjBuYXd5ckpoNjhQQ0I1bURSNTNmCmh1bFhPQ0dTd3ZNN2RwM2RPc3lFQmRlVkw3aTFnbkJNYi9wN05YdTN5WmlWaDlpS3pqaENrZndqL0VsNTZaUHEKYmsvOGtQN0xBdTFvZGJWTkZGSUx5clB6SFBFU3I3N0preHcvKytPTmhtblA2UFBiU3FtRm0rcUVEYWhQanBFZgpscUdaY3BsOEZ0VXBzTG5JK3B4blI5eWU5ZUNpVDVuaDhlTEhobkVFNzFpVE1rb2xrSHdxSm5xV1R3ZlF2b1g5CkM2SERBZ01CQUFHaklUQWZNQjBHQTFVZERnUVdCQlRjOGlDU055NnhSM2M5OU1aYkZUODgzREs1V0RBTkJna3EKaGtpRzl3MEJBUXNGQUFPQ0FRRUFOMEkwZnJDSzdYNGRHRmpGLzFpb3czUUwvbTcwemVIemlQVVFFUUdONXBFMwpyMlZZL0ZHWGdNV0tFSXpHa2hMZW5CUXRxRE5Pbm4vL1JjK0Y2anM1RkNOVXhaT2V4ekl2aElhY0M4YUhzRWpFClRnclI5OWNqUVdsdzFhSVF0YUhkSmVqYXdYcU50YU1FMSt4RlJBQTNCWUF1T3BveW9wQ0NoMXdJaTEvQk84TlQKVmFaancxQU8xd1ZUaTJ3SUtCSUp1Z0N2T0dYZEt3YzBuL0I4bzRRQkpScklRZEJnbzJVNjFBbkMxaWM4b0d3RwpxekN1V0dDenZjam9xNWFNcTliS0YyNHBQaDR3cWZMZnZGdWNsYmFIUlBiSmpxT3l0V3gzczhNV0lvRUNzdlhLCnhIYmJvSnU0c1AwLzRBMGQ4K25OOXI1MU8xalFaWHd0b3hwenFTV2ZlZz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
```

**Certificate XYZ base64 encoded value is not a PEM encoded certificate chain**

The `pem_b64` for each entry in `trusted_certs` ***MUST*** be a PEM encoded certificate or list of pem encoded certificates that form a chain.  This error means
that the base64 encoded data is not a valid certificate.  You can test your input with the openssl command:

```
openssl x509 -inform PEM /path/to/cert.pem
```

If this command outputs your certificate, it's valid.  Do not specify a base64 encoded binary certificate (`DER`).  

**Could not decode certificate XYZ**

An unexpected error with your certificate orccured.  If the error is not explanitory, please [open an issue on GitHub](https://github.com/OpenUnison/openunison-k8s/issues) 

**At least one authentication configuration must be specified or active_directory, oidc, github, or saml**

At least one of the configuration sections in your values.yaml for authenticaiton must be present.  This error means none of them
are.  Uncomment one of the authentication sections.

**You have specified multiple authentication systems : [XXX YYY]**

More then one authentication system has been enabled.  Only one is allowed.  Comment out all but one of the authentication sections.