# 4. Pre-Installation of Jenkins via Helm Chart
Refs: 
- https://www.jenkins.io/doc/book/installing/kubernetes/#install-jenkins-with-helm-v3
- https://github.com/jenkinsci/helm-charts

```sh
helm repo update
helm repo add jenkins https://charts.jenkins.io

helm search repo jenkins/jenkins

# output
NAME            CHART VERSION   APP VERSION     DESCRIPTION                                       
jenkins/jenkins 3.5.17          2.303.1         Jenkins - Build great things at any scale! The ...


# you can output chart file
helm show chart jenkins/jenkins

# you can output default values.yaml, which is the same as https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/values.yaml
helm show values jenkins/jenkins > values.yaml
```


Create K8s namespace `jenkins`
```sh
kubectl create namespace jenkins
```


Create `overrides.yaml`
```yaml
controller:
  # use Docker in Docker jenkins, so that jenkins container can build docker image inside
  statefulSetLabels:
    app: jenkins  # needed for istio kiali dashboard
    version: 2.0.0  # needed for istio kiali dashboard
  serviceLabels:
    app: jenkins  # needed for istio kiali dashboard
    version: 2.0.0  # needed for istio kiali dashboard
  podLabels:
    app: jenkins  # needed for istio kiali dashboard
    version: 2.0.0  # needed for istio kiali dashboard
  JCasC:
    defaultConfig: false # need to disable this otherwise existing plugins and pipeline jobs won't be loaded and you will face "SEVERE	jenkins.InitReactorRunner$1#onTaskFailed: Failed Loading plugin" and "WARNING	c.c.h.p.folder.AbstractFolder#loadChildren:
```