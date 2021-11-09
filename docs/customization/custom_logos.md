# Custom Logos

If you want to use your own logos instead of Tremolo Security's logo, you'll need to create 3 PNG files:

| Filename | Dimmensions | Use |
| -------- | ----------- | --- |
| ts_logo.png | 294w x 103h | Login and error pages |
| logo-desktop.png | 144w x 59h | Small logo in upper left hand corner on larger/desktop windows |
| logo-mobile.png | 646w x 220h | Large logo used when the window is compressed or on a mobile screen |

Create a `ConfigMap` in the `openunison` namespace:

```
kubectl create ConfigMap custom-logos -n openunison --from-file=/path/to/logos
```

Update your values.yaml file by adding `openunison.html.logosConfigMap` with the name of the `ConfigMap`:

```
openunison:
  .
  .
  .
  html:
    image: docker.io/tremolosecurity/openunison-k8s-html
    logosConfigMap: custom-logos
```

Finally, upgrade your helm deployments:

```
helm upgrade orchestra tremolo/orchestra -n openunison -f /path/to/values.yaml
helm upgrade orchestra-login-portal tremolo/orchestra-login-portal -n openunison -f /path/to/values.yaml
```

Once all containers are running again, your logos will appear instead of Tremolo Security's logos.