# 5. Create PVC for Jenkins
Refs: 
- https://www.jenkins.io/doc/book/installing/kubernetes/#create-a-persistent-volume
- https://github.com/jenkinsci/helm-charts/tree/main/charts/jenkins#persistence


Make sure `pvc.yaml` refers to the matching existing `storageClassName: efs-sc`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-jenkins-claim
  namespace: jenkins # <----- make sure to create PVC in jenkins ns
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

Apply
```sh
kubectl apply -f pvc_jenkins.yaml

# output
$ kubectl get pvc

NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
efs-jenkins-claim   Bound    pvc-61d67528-cd38-4b85-a97b-2671cccaf730   5Gi        RWX            efs-sc         66s
```


Update Jenkins helm `overrides.yaml`
```yaml
controller:
  # use Docker in Docker jenkins, so that jenkins container can build docker image inside
  statefulSetLabels:
    app: jenkins  # needed for istio
    version: 2.0.0  # needed for istio
  serviceLabels:
    app: jenkins  # needed for istio
    version: 2.0.0  # needed for istio
  podLabels:
    app: jenkins  # needed for istio
    version: 2.0.0  # needed for istio
  JCasC:
    defaultConfig: false # need to disable this otherwise existing plugins and pipeline jobs won't be loaded and you will face "SEVERE	jenkins.InitReactorRunner$1#onTaskFailed: Failed Loading plugin" and "WARNING	c.c.h.p.folder.AbstractFolder#loadChildren:

persistence:
  existingClaim: efs-jenkins-claim
  storageClass: efs-sc
```
