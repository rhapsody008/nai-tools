# Tools Onboarding using Flux

Currently deployed to the workspace namespace of the workload cluster. Change create the namespace if needed

## Prerequisite

Cloudflare specific: create the secret before the deployment. Create the namespace if not exist.

```
kubectl create namespace cloudflared

kubectl create secret generic cloudflared-credentials \
  --namespace=cloudflared \
  --from-file=credentials.json=czxxz.json \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Bootstrap

Bootstrap Flux using the tools-flux.yaml. Do once for the cluster.

```
kubectl apply tools-flux.yaml
```

## Adding new tools
1. create a new dir with necessary files. one kustomization.yaml and other k8s resources.
2. create <tool>.yaml in ./src dir. - Commit the change and this tool will be auto synced by tools-flux.yaml.

