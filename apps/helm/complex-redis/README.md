# complex-redis

This shows how you can integrate argocd-vault-plugin with Helm chart via pre-existing secrets deployed via AVP. Rather than injecting placeholders into the Helm values, and having AVP process them after rendering the chart, you can simply deploy the "extra" secrets a Helm charts needs _before_ deploying the chart via the app-of-apps pattern and sync-waves.

We create an `app.yaml` that will deploy 2 child Argo apps from `child-apps` - this is the app of apps pattern. 

The first child app to be deployed will be the `redis-manifests` app which will contain the secret we need to exist in the cluster before installing the chart. It will be deployed first because it's `annotation` value is lower than the the `redis` app.

The second child app, `redis`, just installs the Redis Helm chart from upstream. We can use the `helm`-native Argo integration here since the secrets and placeholders are handled by the `redis-manifests` app.

The `extra-manifests` folder just contains the YAMLs that comprise the `redis-manifests` app - in our case, a secret we want to process via AVP.