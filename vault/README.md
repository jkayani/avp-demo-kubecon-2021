# Configuring Vault to use Kubernetes auth

[Previously](../argocd/overlays) we configured AVP within our repo-server to use Kubernetes authentication (service account token auth). To configure the Vault side of this, we have to:

- Create a policy permitting `read`ing secrets from the KV-V2 secret path (`secret`)

- Enable Kubernetes auth in Vault, configuring Vault to use our Kubernetes API server to verify our SA token. We disable issuer validation because of a change in Kubernetes 1.21.

- Create an `argocd` Vault role, granted to the `default` SA in the `default` namespace, and give it the policy we created earlier
