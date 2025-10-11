now in order to see the function of admission controller mutating i implemented a pvc.yaml with no storage class mentioned. following was the manifest->
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}

```


and then when i checked->
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"myclaim","namespace":"default"},"spec":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"8Gi"}},"selector":{"matchExpressions":[{"key":"environment","operator":"In","values":["dev"]}],"matchLabels":{"release":"stable"}},"volumeMode":"Filesystem"}}
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: k8s.io/minikube-hostpath
    volume.kubernetes.io/storage-provisioner: k8s.io/minikube-hostpath
  creationTimestamp: "2025-10-11T06:39:07Z"
  finalizers:
  - kubernetes.io/pvc-protection
  name: myclaim
  namespace: default
  resourceVersion: "863"
  uid: 6e8616e2-15ec-483b-906e-2cb987df799e
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  selector:
    matchExpressions:
    - key: environment
      operator: In
      values:
      - dev
    matchLabels:
      release: stable
  storageClassName: standard
  volumeMode: Filesystem
  volumeName: pvc-6e8616e2-15ec-483b-906e-2cb987df799e
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 8Gi
  phase: Bound
```


on carefully looking i got->
```yaml
 storageClassName: standard
  volumeMode: Filesystem
  volumeName: pvc-6e8616e2-15ec-483b-906e-2cb987df799e
```
and here earlier the storage class wasnot available and this concept is known as mutation by admission controller. and why it happened bcz i had a default storage class admission controller check admission-controller.md
