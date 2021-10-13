# Kubecon 2021 Demo: argocd-vault-plugin

## Outline

1. Deploy ArgoCD and Hashicorp Vault

    - Install argocd-vault-plugin (AVP)

    - Enable Kubernetes authentication

1. Deploy a simple Git-based Argo CD application

1. Deploy a Helm chart through Argo CD

Details for all manifests applied to our clusters are available in `README` files in the manifests containing folder.

## Deploy Argo CD and Vault

### Deploy Argo CD
```sh
kustomize build argocd/overlays | kubectl apply -f -

kubectl port-forward svc/argocd-server 8080:80 &>/dev/null &

kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
```

### Deploy Vault
```sh
kubectl apply -f vault/app.yaml
```

## Deploy a Git app (Nginx) with argocd-vault-plugin

### Create a set of YAMLs for our app
```sh
cat > apps/git/nginx/manifests/all.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nginx
data:  
  nginx.conf: |
    events {}
    env MY_SECRET;
    http {
        server {
            listen 8080;
            location / {
                set_by_lua \$my_secret 'return os.getenv("MY_SECRET")';
                return 200 \$my_secret;
            }
        }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: my-nginx
  annotations:

    # Our special annotation to tell AVP where the secrets are
    avp.kubernetes.io/path: "secret/data/my-nginx"
stringData:  
  MY_SECRET: <password>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: openresty/openresty:1.19.9.1-0-alpine
        ports:
        - containerPort: 8080
        envFrom:
        - secretRef:
            name: my-nginx
        volumeMounts:
        - name: nginx-conf
          mountPath: /usr/local/openresty/nginx/conf/
      volumes:
        - name: nginx-conf
          configMap:
            name: my-nginx
EOF

git add apps/git/nginx/all.yaml
git commit -m "demo: Creating the files for my ArgoCD app"
git push 
```

### Create a secret in our Secret Manager
```sh
  kubectl port-forward svc/vault 8200 &>/dev/null &
  export VAULT_ADDR=http://localhost:8200
  export VAULT_TOKEN=root
  vault kv put secret/my-nginx password="secret-password"
  vault kv get secret/my-nginx
```

### Create our Argo app
```sh
cat > apps/git/nginx/app.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-nginx
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc 
  project: default
  source:
    repoURL: 'https://github.com/jkayani/avp-demo-kubecon-2021'
    targetRevision: HEAD
    path: apps/git/nginx/manifests

    # We're telling Argo CD to use our plugin to deploy the manifests
    plugin:
      name: argocd-vault
EOF

kubectl apply -f apps/git/nginx/app.yaml

kubectl port-forward deployment/my-nginx 8081:8080
```

### Modify our secret in our Secret Manager
```sh
  vault kv put secret/my-nginx password="secret-password-edited"
  vault kv get secret/my-nginx
```

### Hard-refresh our Argo app and restart our deployment to see the change
```sh
  argocd app diff example-git-app --hard-refresh
  kubectl rollout restart deployment/my-nginx
```

## Deploy a Helm chart (Redis) with argocd-vault-plugin

### Create a secret in our Secret Manager
```sh
vault kv put secret/my-redis password="shhh"
vault kv get secret/my-redis
```

### Create our Argo app
```sh
cat > apps/helm/simple-redis/app.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-redis
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc 
  project: default
  source:
    repoURL: https://github.com/bitnami/charts/
    targetRevision: HEAD
    path: bitnami/redis
    plugin:
      name: argocd-vault-helm
      env:

        # These are the arguments we pass to "helm template"
        - name: helm_args
          value: |
            --set architecture=standalone
            --set auth.enabled=true
            --set global.redis.password=<path:secret/data/my-redis#password>
EOF

kubectl apply -f apps/helm/simple-redis/app.yaml

kubectl port-forward svc/my-redis-master 6379

redis-cli --askpass
```
