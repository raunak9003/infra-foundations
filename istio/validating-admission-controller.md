now to test it i used a resource quota manifest->

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-demo
spec:
  hard:
    cpu: "1"
    memory: 2Gi

```

then i applied manifest of a pod->


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: demo
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
```

now on implementing this pod manifest observed this error->

```yaml
raunak_jhunjhunwala_9003@Raunaks-MacBook-Pro istio % kubectl apply -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "demo-pod" is forbidden: exceeded quota: quota-demo, requested: memory=10Gi, used: memory=0, limited: memory=2Gi
raunak_jhunjhunwala_9003@Raunaks-MacBook-Pro istio % 
```

now why it happened?
bcz i had a resourcequota admission controller which validated and rejected it
