#k8s 

k8s강의에서 factorial-app 서비스에서 사용한 yaml은 다음이다.

\<pvc yaml>
```yaml
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: cache-log-storage-claim  
  namespace: factorial  
spec:  
  storageClassName: local-storage  
  accessModes:  
    - ReadWriteOnce  
  resources:  
    requests:  
      storage: 100Mi
```

\<storage yaml>
```yaml
apiVersion: storage.k8s.io/v1  
kind: StorageClass  
metadata:  
  name: local-storage  
provisioner: rancher.io/local-path  
volumeBindingMode: WaitForFirstConsumer
```

우선 pvc, storage에 대한 개념부터 가볍게 잡아보자.
### PVC
persistent volume claim

간단하게 위 yaml에 대한 궁금증 부터 나열한 뒤, 1개씩 알아보자.
