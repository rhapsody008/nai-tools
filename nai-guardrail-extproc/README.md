# Steps after container image available

1. create secret 

```
kubectl create secret generic guardrail-extproc-creds \
  --namespace nai-system \
  --from-literal=GUARDRAIL_API_KEY='<your real key>'
```

