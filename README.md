# Tools Onboarding using Flux

Currently deployed to the workspace namespace of the workload cluster. Change create the namespace if needed

## Bootstrap

Bootstrap Flux using the tools-flux.yaml. Do once for the cluster.

```
kubectl apply tools-flux.yaml
```

## Adding new tools
1. create a new dir with necessary files. one kustomization.yaml and other k8s resources.
2. create <tool>.yaml in ./src dir. - Commit the change and this tool will be auto synced by tools-flux.yaml.

