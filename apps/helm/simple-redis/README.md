# simple-redis

This shows how you can integrate argocd-vault-plugin with a Helm chart with no extra YAMLs.

`app.yaml` is an Argo app that uses the Redis Helm chart as the set of manifests, and then uses AVP to process and deploy the Helm chart. We specifically inject our AVP placeholders into the Helm values, and then have Argo run `helm template | argocd-vault-plugin generate -` to render and process the chart templates.

We're using [inline-path placeholders](https://ibm.github.io/argocd-vault-plugin/v1.4.0/howitworks/#inline-path-placeholders) to interpolate the value of the global Redis password. Our placeholder `<path:secret/data/my-redis#password>` says to take the `my-redis` secret and interpolate its `password` key for the value of that Helm chart parameter.
