Command for delete error pods

```bash
kubectl get pods | grep Evicted | awk '{print $1}' | xargs kubectl delete pod
```

```bash
kubectl --namespace=production get pods -a | grep Evicted | awk '{print $1}' | xargs kubectl --namespace=production delete pod -o name
```
