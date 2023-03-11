# 13. How to Limit Concurrent Jenkins Jobs

Manage Jenkins > Manage Nodes and Clouds > Configure Clouds > Kubernetes Cloud Details > Concurrency Limit

![alt text](../imgs/jenkins_concurrent_worker_pods.png "")


NOTE: using [Throttle Concurrent Builds Plugin](https://plugins.jenkins.io/throttle-concurrents/) didn't work for k8s.