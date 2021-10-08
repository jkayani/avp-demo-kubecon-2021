# Installing AVP with Argo CD

2 main things:

- Modify repo-server deployment to use an initContainer to download our AVP binary: <https://ibm.github.io/argocd-vault-plugin/v1.4.0/installation/#initcontainer>

  - Configure AVP (which runs within the repo-server container) via env variables on the repo-server: <https://ibm.github.io/argocd-vault-plugin/v1.4.0/config/#environment-variables>

  - Mount our service account token

- Register 2 custom plugins using AVP 

  - The first one is for plain manifests

  - Second is for using AVP with Helm charts: <https://ibm.github.io/argocd-vault-plugin/v1.4.0/usage/#with-helm>