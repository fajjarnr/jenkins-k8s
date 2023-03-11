# 6. Install Jenkins via Helm Chart

Install Helm chart using .tgz file 
```sh
# using compressed file instead of jenkins/jenkins to lock down the version
helm install jenkins jenkins/jenkins \
    -n jenkins \
    -f overrides.yaml

# output
helm status jenkins -n jenkins

NAME: jenkins
LAST DEPLOYED: Sat Sep 11 21:27:16 2021
NAMESPACE: jenkins
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  echo http://127.0.0.1:8080
  kubectl --namespace jenkins port-forward svc/jenkins 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

It takes about 3-5 mins before it becomes running

Check pod's status
```sh
watch -t 'kubectl get pod -n jenkins'

NAME        READY   STATUS    RESTARTS   AGE
jenkins-0   2/2     Running   0          34m

# get logs
kubectl logs jenkins-0 -n jenkins -c init
```


Access Jenkins from a browser
```
localhost:8080
```


Make sure Jenkins config files are stored on EFS mount point
```sh
$ ssh-add -k ~/.ssh/aws-demo/eks-demo-workers.pem
$ ssh -A ec2-user@13.212.234.2
Last login: Sat Sep 11 13:51:44 2021 from 49.228.39.156

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-192-168-65-222 ~]$ df -h
Filesystem                                      Size  Used Avail Use% Mounted on
devtmpfs                                        3.9G     0  3.9G   0% /dev
tmpfs                                           3.9G     0  3.9G   0% /dev/shm
tmpfs                                           3.9G  1.3M  3.9G   1% /run
tmpfs                                           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/nvme0n1p1                                   80G  5.1G   75G   7% /
fs-cdee558d.efs.ap-southeast-1.amazonaws.com:/  8.0E  358M  8.0E   1% /mnt/efs  # <---- EFS is mounted at /mnt/efs
tmpfs                                           786M     0  786M   0% /run/user/1000

# jenkins HOME (/var/lib/jenkins) is at /mnt/efs/dynamic_provisioning/pvc-xxxxx
[ec2-user@ip-192-168-65-222 ~]$ sudo ls /mnt/efs/dynamic_provisioning/pvc-8becabff-4a9a-4fbd-be41-ab44c8122a9f/
casc_configs                                                 nodeMonitors.xml
config.xml                                                   nodes
copy_reference_file.log                                      plugins
hudson.model.UpdateCenter.xml                                plugins.txt
hudson.plugins.git.GitTool.xml                               queue.xml.bak
identity.key.enc                                             secret.key
jenkins.install.InstallUtil.lastExecVersion                  secret.key.not-so-secret
jenkins.install.UpgradeWizard.state                          secrets
jenkins.security.apitoken.ApiTokenPropertyConfiguration.xml  updates
jenkins.security.QueueItemAuthenticatorConfiguration.xml     userContent
jenkins.security.UpdateSiteWarningsConfiguration.xml         users
jenkins.telemetry.Correlator.xml                             war
jobs                                                         workflow-libs
logs
```