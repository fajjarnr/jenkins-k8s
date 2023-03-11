# 2. AWS EKS Cluster Setup to mock production-like infra



## 2.1 Install CLIs (aws, eksctl, helm)

### AWS CLI
Ref: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

```bash
# for Mac
# install homebrew first 
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

# Mac
brew install awscli

# Windows: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html

aws --version
aws configure

aws sts get-caller-identity
```

### aws-iam-authenticator (if aws cli is 1.16.156 or earlier)
```bash
# Mac
brew install aws-iam-authenticator

# Windows
# install chocolatey first: https://chocolatey.org/install
choco install -y aws-iam-authenticator
```

### kubectl
Ref: https://kubernetes.io/docs/tasks/tools/install-kubectl/

```bash
# Mac
brew install kubectl 

# Windows
choco install kubernetes-cli

kubectl version
```

### eksctl
Ref: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html

```bash
# Mac
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# or upgrade
brew upgrade eksctl && brew link --overwrite eksctl

eksctl version

# Windows: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
# install eskctl from chocolatey
chocolatey install -y eksctl 

eksctl version
```

## Helm v3
Ref: https://helm.sh/docs/intro/install/

```bash
# Mac
brew install helm

# Windows
choco install kubernetes-helm

# verify
helm version
```

### Create ssh key for EKS worker nodes
```bash
ssh-keygen
eks-demo-workers.pem
```




## 2.2 Setup EKS cluster with eksctl (so you don't need to manually create VPC)

`eksctl` tool will create K8s Control Plane (master nodes, etcd, API server, etc), worker nodes, VPC, Security Groups, Subnets, Routes, Internet Gateway, etc.
```bash
# EKS not supported in us-west-1

export AWS_PROFILE=aws-demo
export AWS_REGION="ap-southeast-1"  # NOTE: change this to your own
CLUSTER_NAME="eks-from-eksctl"

eksctl create cluster \
    --name ${CLUSTER_NAME} \
    --version 1.21 \
    --region ${AWS_REGION} \
    --nodegroup-name workers \
    --node-type t3.large \
    --nodes 1 \
    --nodes-min 1 \
    --nodes-max 2 \
    --ssh-access \
    --ssh-public-key ~/.ssh/aws-demo/eks-demo-workers.pem.pub \
    --managed
```

Output
```bash
[ℹ]  eksctl version 0.66.0
[ℹ]  using region ap-southeast-1
[ℹ]  setting availability zones to [ap-southeast-1b ap-southeast-1a ap-southeast-1c]
[ℹ]  subnets for ap-southeast-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for ap-southeast-1a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for ap-southeast-1c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using SSH public key "/Users/USERNAME/.ssh/eks-demo-workers.pem.pub" as "eksctl-eks-from-eksctl-nodegroup-workers-51:34:9d:9e:0f:87:a5:dc:0c:9f:b9:0c:29:5a:0b:51" 
[ℹ]  using Kubernetes version 1.21
[ℹ]  creating EKS cluster "eks-from-eksctl" in "ap-southeast-1" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-1 --cluster=eks-from-eksctl'
[ℹ]  CloudWatch logging will not be enabled for cluster "eks-from-eksctl" in "ap-southeast-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=ap-southeast-1 --cluster=eks-from-eksctl'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-from-eksctl" in "ap-southeast-1"
[ℹ]  2 sequential tasks: { create cluster control plane "eks-from-eksctl", 2 sequential sub-tasks: { no tasks, create managed nodegroup "workers" } }
[ℹ]  building cluster stack "eksctl-eks-from-eksctl-cluster"
[ℹ]  deploying stack "eksctl-eks-from-eksctl-cluster"
[ℹ]  building managed nodegroup stack "eksctl-eks-from-eksctl-nodegroup-workers"
[ℹ]  deploying stack "eksctl-eks-from-eksctl-nodegroup-workers"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/Users/USERNAME/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "eks-from-eksctl" have been created
[ℹ]  nodegroup "workers" has 2 node(s)
[ℹ]  node "ip-192-168-20-213.ap-southeast-1.compute.internal" is ready
[ℹ]  node "ip-192-168-39-97.ap-southeast-1.compute.internal" is ready
[ℹ]  waiting for at least 1 node(s) to become ready in "workers"
[ℹ]  nodegroup "workers" has 2 node(s)
[ℹ]  node "ip-192-168-20-213.ap-southeast-1.compute.internal" is ready
[ℹ]  node "ip-192-168-39-97.ap-southeast-1.compute.internal" is ready
[ℹ]  kubectl command should work with "/Users/USERNAME/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eks-from-eksctl" in "ap-southeast-1" region is ready
```

Once you have created a cluster, you will find that cluster credentials were added in ~/.kube/config

```bash
# get info about cluster resources
aws eks describe-cluster --name eks-from-eksctl --region ap-southeast-1
```

Output
```json
{
    "cluster": {
        "name": "eks-from-eksctl",
        "arn": "arn:aws:eks:ap-southeast-1:266981300450:cluster/eks-from-eksctl",
        "createdAt": "2021-09-11T17:25:27.181000+07:00",
        "version": "1.21",
        "endpoint": "https://864E4B20969F9E54A143625B9E0E2FFA.gr7.ap-southeast-1.eks.amazonaws.com",
        "roleArn": "arn:aws:iam::266981300450:role/eksctl-eks-from-eksctl-cluster-ServiceRole-JAHBOFUR4EK1",
        "resourcesVpcConfig": {
            "subnetIds": [
                "subnet-0028ba5b800873cf9",
                "subnet-07c3896be602e9fa3",
                "subnet-02682407940c3847a",
                "subnet-0cb09e26fc1b1f0c5",
                "subnet-0facd6d85882c13a8",
                "subnet-033515a0193e46835"
            ],
            "securityGroupIds": [
                "sg-038bf2e1bff049643"
            ],
            "clusterSecurityGroupId": "sg-064dc4e915344358a",
            "vpcId": "vpc-081f83f675c97c381",
            "endpointPublicAccess": true,
            "endpointPrivateAccess": false,
            "publicAccessCidrs": [
                "0.0.0.0/0"
            ]
        },
        "kubernetesNetworkConfig": {
            "serviceIpv4Cidr": "10.100.0.0/16"
        },
        "logging": {
            "clusterLogging": [
                {
                    "types": [
                        "api",
                        "audit",
                        "authenticator",
                        "controllerManager",
                        "scheduler"
                    ],
                    "enabled": false
                }
            ]
        },
        "identity": {
            "oidc": {
                "issuer": "https://oidc.eks.ap-southeast-1.amazonaws.com/id/864E4B20969F9E54A143625B9E0E2FFA"
            }
        },
        "status": "ACTIVE",
        "certificateAuthority": {
            "data": "xxx"
        },
        "platformVersion": "eks.2",
        "tags": {}
    }
}
```

```bash
# get services
kubectl get svc
```

Output shows the default `kubernetes` service, which is the API server in master node
```bash
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   38m
```