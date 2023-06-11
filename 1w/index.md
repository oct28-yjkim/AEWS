---
title: "eks 를 배포 해봅시다. "
date: 2023-04-29T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 1w
    identifier: 1w
    parent: cloudnet-aews
    weight: 30
author:
  name: gopher
  image: /images/author/eks-e.png
math: true
---

cloudnet study 에서 aws eks 스터디 모집 요강이 있었으며 
이런저런 사정에 의해서 스터디 신청을 할 수 있게 되어서 신청을 하여 진행하게 되었다. 

이번에는 aws 에서 k8s 를 kops 와 같은 도구가 아닌 eksctl 을 이용하여 eks 를 배포, 실습, 학습 
하는 과정을 거치게 될듯 하다. 

그중 1장은 
* client host 배포 
* eks 를 배포 하여서 각 클러스터 상태 확인 
* simple application 을 배포 하여서 상태 확인 
* bottlerocket 를 배포 하여서 위  amilinux2 와는 어디가 다른것인지 
등으로 4가지의 챕터를 진행 할 예정이다. 

```markdown 
모든 자료들은 http://www.cloudneta.net 에서 찾을수 있다.
```

---
## client host 배포 

배포 하기전에 aws 의 keypair 을 구성 해주어야한다. 

myeks-1week.yaml 파일내에서는 다음과 같은 작업을 수행해주는 cloudformation manifest 이다. 

* AWS 내 eks 를 배포하기 위한 객체들
  * VPC
  * public subnet 1,2 
  * private subnet 1,2
  * IGW
  * subnet Route 관련 객체들 : routetable, route, routetable association
  * ec2 : eksctl host 
  * ec2 내의 사용자 도구들 : kubectl, eksctl, awscli, yh, krew, kubeps-1, docker
  * 기타 호스트 설정도구 : ntp, vim, bash 구성, ssh keypair 

```sh 
# yaml 파일 다운로드
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/myeks-1week.yaml

# 배포
aws cloudformation deploy --template-file ./myeks-1week.yaml \
     --stack-name yjkim-eks --parameter-overrides KeyName=lala-yjkim SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32 --region ap-northeast-3

# ec2 에 SSH 접속
ssh -i ~/.ssh/yjkim/lala-yjkim.pem ec2-user@$(aws cloudformation describe-stacks --stack-name yjkim-eks --query 'Stacks[*].Outputs[0].OutputValue' --output text)

```

---

## eksctl 을 이용하여 eks 배포 실습 


### 환경 변수 구성 

```sh 
# client host 에도 aws configure 를 진행하여 aws cli 를 이용할 수 있도록 구성 
aws configure

# 자격 구성 적용 확인 : 노드 IP 확인
aws ec2 describe-instances

# EKS 배포할 VPC 정보 확인
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=$CLUSTER_NAME-VPC" | jq
echo "export VPCID=$VPCID" >> /etc/profile
echo VPCID

# EKS 배포할 VPC에 속한 Subnet 정보 확인
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPCID" --output json | jq
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPCID" --output yaml | yh

## 퍼블릭 서브넷 ID 확인
aws ec2 describe-subnets --filters Name=tag:Name,Values="$CLUSTER_NAME-PublicSubnet1" | jq
aws ec2 describe-subnets --filters Name=tag:Name,Values="$CLUSTER_NAME-PublicSubnet1" --query "Subnets[0].[SubnetId]" --output text
export PubSubnet1=$(aws ec2 describe-subnets --filters Name=tag:Name,Values="$CLUSTER_NAME-PublicSubnet1" --query "Subnets[0].[SubnetId]" --output text)
export PubSubnet2=$(aws ec2 describe-subnets --filters Name=tag:Name,Values="$CLUSTER_NAME-PublicSubnet2" --query "Subnets[0].[SubnetId]" --output text)
echo "export PubSubnet1=$PubSubnet1" >> /etc/profile
echo "export PubSubnet2=$PubSubnet2" >> /etc/profile
echo $PubSubnet1
echo $PubSubnet2
```


### 구성 예시 

* eksctl 에서는 create cluster 명령을 통하여 eks 클러스터를 생성 할 수 있으며 내부적으로는 cloudformation 을 이용하여 배포 한다. 
* dryrun 명령을 이용하여 manifest 가 어떻게 구성되는지 확인할 예정이다. 

```sh 
# 기본적으로 가용존 생성 예시 : 3개 
$ eksctl create cluster --name myeks --region=ap-northeast-3 --without-nodegroup --dry-run | yh
apiVersion: eksctl.io/v1alpha5
availabilityZones:
- ap-northeast-3c
- ap-northeast-3a
- ap-northeast-3b
cloudWatch:
  clusterLogging: {}
iam:
  vpcResourceControllerPolicy: true
  withOIDC: false
kind: ClusterConfig
kubernetesNetworkConfig:
  ipFamily: IPv4
metadata:
  name: myeks
  region: ap-northeast-3
  version: "1.25"
privateCluster:
  enabled: false
  skipEndpointCreation: false
vpc:
  autoAllocateIPv6: false
  cidr: 192.168.0.0/16
  clusterEndpoints:
    privateAccess: false
    publicAccess: true
  manageSharedNodeSecurityGroupRules: true
  nat:
    gateway: Single


# 가용존을 명시 하여 2개의 존으로 구성 예시
$ eksctl create cluster --name myeks --region=ap-northeast-3 --without-nodegroup --zones=ap-northeast-2a,ap-northeast-2c --dry-run | yh
apiVersion: eksctl.io/v1alpha5
availabilityZones:
- ap-northeast-2a
- ap-northeast-2c
cloudWatch:
  clusterLogging: {}
iam:
  vpcResourceControllerPolicy: true
  withOIDC: false
kind: ClusterConfig
kubernetesNetworkConfig:
  ipFamily: IPv4
metadata:
  name: myeks
  region: ap-northeast-3
  version: "1.25"
privateCluster:
  enabled: false
  skipEndpointCreation: false
vpc:
  autoAllocateIPv6: false
  cidr: 192.168.0.0/16
  clusterEndpoints:
    privateAccess: false
    publicAccess: true
  manageSharedNodeSecurityGroupRules: true
  nat:
    gateway: Single

# nodegroup, nodetype, volume, vpc-cidr 명시 
$ eksctl create cluster --name eks-ami --region=ap-northeast-2 --nodegroup-name=mynodegroup --node-type=t3.medium --node-volume-size=30 \
> --zones=ap-northeast-2a,ap-northeast-2c --vpc-cidr=172.20.0.0/16 --dry-run | yh
apiVersion: eksctl.io/v1alpha5
availabilityZones:
- ap-northeast-2a
- ap-northeast-2c
...
managedNodeGroups:
- amiFamily: AmazonLinux2
  desiredCapacity: 2
  disableIMDSv1: false
  disablePodIMDS: false
  iam:
    withAddonPolicies:
      albIngress: false
      appMesh: false
      appMeshPreview: false
      autoScaler: false
      awsLoadBalancerController: false
      certManager: false
      cloudWatch: false
      ebs: false
      efs: false
      externalDNS: false
      fsx: false
      imageBuilder: false
      xRay: false
  instanceSelector: {}
  instanceType: t3.medium  # instance type
  labels:
    alpha.eksctl.io/cluster-name: eks-ami
    alpha.eksctl.io/nodegroup-name: mynodegroup # node group 
  maxSize: 2
  minSize: 2
  name: mynodegroup
  privateNetworking: false
  releaseVersion: ""
  securityGroups:
    withLocal: null
    withShared: null
  ssh:
    allow: false  # ssh access 
    publicKeyPath: ""
  tags:
    alpha.eksctl.io/nodegroup-name: mynodegroup
    alpha.eksctl.io/nodegroup-type: managed
  volumeIOPS: 3000
  volumeSize: 30 # volume size 
  volumeThroughput: 125
  volumeType: gp3
metadata:
  name: eks-ami
  region: ap-northeast-2
  version: "1.25"
privateCluster:
  enabled: false
  skipEndpointCreation: false
vpc:
  autoAllocateIPv6: false
  cidr: 172.20.0.0/16
  clusterEndpoints:
    privateAccess: false
    publicAccess: true
  manageSharedNodeSecurityGroupRules: true
  nat:
    gateway: Single

```

## ekscli 를 이용하여 eks ami cluster 배포 

```sh 
# 위에서 작성한 변수 확인 
$ echo $AWS_DEFAULT_REGION
ap-northeast-3
$ echo $CLUSTER_NAME
myeks
$ echo $VPCID
vpc-08eb4eec4bd25a80d
$ echo $PubSubnet1,$PubSubnet2
subnet-01497e12118abcbe2,subnet-0ef48df128ded06ad
$ CLUSTER_NAME=eks-ami
$ echo $CLUSTER_NAME
eks-ami

# dryrun 을 통해 manifest 를 확인, 대충 불필요한 값은 지운 상태이다. 
$ eksctl create cluster --name $CLUSTER_NAME --region=$AWS_DEFAULT_REGION --nodegroup-name=$CLUSTER_NAME-nodegroup --node-type=t3.medium \
> --node-volume-size=30 --vpc-public-subnets "$PubSubnet1,$PubSubnet2" --version 1.24 --ssh-access --external-dns-access --dry-run | yh
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
kubernetesNetworkConfig:
  ipFamily: IPv4
managedNodeGroups:
- amiFamily: AmazonLinux2
  desiredCapacity: 2
  instanceSelector: {}
  instanceType: t3.medium
  labels:
    alpha.eksctl.io/cluster-name: eks-ami
    alpha.eksctl.io/nodegroup-name: eks-ami-nodegroup
  maxSize: 2
  minSize: 2
  name: eks-ami-nodegroup
  ssh:
    allow: true
    publicKeyPath: ~/.ssh/id_rsa.pub
  tags:
    alpha.eksctl.io/nodegroup-name: eks-ami-nodegroup
    alpha.eksctl.io/nodegroup-type: managed
  volumeIOPS: 3000
  volumeSize: 30
  volumeThroughput: 125
  volumeType: gp3
metadata:
  name: eks-ami
  region: ap-northeast-3
  version: "1.24"
privateCluster:
  enabled: false
  skipEndpointCreation: false
vpc:
  autoAllocateIPv6: false
  cidr: 192.168.0.0/16
  clusterEndpoints:
    publicAccess: true
  id: vpc-08eb4eec4bd25a80d
  manageSharedNodeSecurityGroupRules: true
  nat:
    gateway: Disable
  subnets:
    public:
      ap-northeast-3a:
        az: ap-northeast-3a
        cidr: 192.168.1.0/24
        id: subnet-01497e12118abcbe2
      ap-northeast-3c:
        az: ap-northeast-3c
        cidr: 192.168.2.0/24
        id: subnet-0ef48df128ded06ad

# cluster 생성 
eksctl create cluster --name $CLUSTER_NAME --region=$AWS_DEFAULT_REGION --nodegroup-name=$CLUSTER_NAME-nodegroup --node-type=t3.medium \
--node-volume-size=30 --vpc-public-subnets "$PubSubnet1,$PubSubnet2" --version 1.24 --ssh-access --external-dns-access --verbose 4

```


### 클러스터 정보 확인 

```sh 

# 생성된 클러스터 조회 
aws eks list-clusters
{
    "clusters": [
        "eks-ami"
    ]
}

# 생성된 노드 그룹 조회  
aws eks list-nodegroups --cluster-name eks-ami
{
    "nodegroups": [
        "eks-ami-nodegroup"
    ]
}

# 노드 그룹으로 autoscaling group 을 조회 
aws autoscaling describe-auto-scaling-groups
{
    "AutoScalingGroups": [
        {
            "AutoScalingGroupName": "eks-eks-ami-nodegroup-00c3e667-fc25-5d32-cda1-5e45fcab08c5",
            "MixedInstancesPolicy": {
                "LaunchTemplate": {
                    "LaunchTemplateSpecification": {
                        "LaunchTemplateId": "lt-027aecc33f45fe271",
                        "LaunchTemplateName": "eks-00c3e667-fc25-5d32-cda1-5e45fcab08c5",
                        "Version": "1"
                    },
                    "Overrides": [
                        {
                            "InstanceType": "t3.medium"
                        }
                    ]
                },
                "InstancesDistribution": {
                    "OnDemandAllocationStrategy": "prioritized",
                    "OnDemandBaseCapacity": 0,
                    "OnDemandPercentageAboveBaseCapacity": 100,
                    "SpotAllocationStrategy": "lowest-price",
                    "SpotInstancePools": 2
                }
            },
            "MinSize": 2, # instance size 
            "MaxSize": 2,
            "DesiredCapacity": 2,
            "DefaultCooldown": 300,
            "AvailabilityZones": [ # nodegroup 가용존 
                "ap-northeast-3c",
                "ap-northeast-3a"
            ],
            "Instances": [
                {
                    "InstanceId": "i-03d33944a3869ba85",
                    "InstanceType": "t3.medium",
                    "AvailabilityZone": "ap-northeast-3c",
                    "LifecycleState": "InService",
                    "HealthStatus": "Healthy",
                    "LaunchTemplate": {
                        "LaunchTemplateId": "lt-027aecc33f45fe271",
                        "LaunchTemplateName": "eks-00c3e667-fc25-5d32-cda1-5e45fcab08c5",
                        "Version": "1"
                    },
                    "ProtectedFromScaleIn": false
                },
                {
                    "InstanceId": "i-0ea68d68d7dfa83a7",
                    "InstanceType": "t3.medium",
                    "AvailabilityZone": "ap-northeast-3a",
                    "LifecycleState": "InService",
                    "HealthStatus": "Healthy",
                    "LaunchTemplate": {
                        "LaunchTemplateId": "lt-027aecc33f45fe271",
                        "LaunchTemplateName": "eks-00c3e667-fc25-5d32-cda1-5e45fcab08c5",
                        "Version": "1"
                    },
                    "ProtectedFromScaleIn": false
                }
            ],
            "CreatedTime": "2023-04-29T11:40:40.642000+00:00",
            "SuspendedProcesses": [],
            "VPCZoneIdentifier": "subnet-01497e12118abcbe2,subnet-0ef48df128ded06ad"
    ]
}

# 생성된 클러스터 endpoint 조회 
$ aws eks describe-cluster --name $CLUSTER_NAME | jq -r .cluster.endpoint
https://E2EFEBC45260D71D72BEA7511606C59A.gr7.ap-northeast-3.eks.amazonaws.com

# eks API 접속 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# curl -k -s $(aws eks describe-cluster --name $CLUSTER_NAME | jq -r .cluster.endpoint)
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}(yjkim@eks-ami:N/A) [root@myeks-host ~]# curl -k -s $(aws eks describe-cluster --name $CLUSTER_NAME | jq -r .cluster.endpoint)/version | jq
{
  "major": "1",
  "minor": "24+",
  "gitVersion": "v1.24.12-eks-ec5523e",
  "gitCommit": "3939bb9475d7f05c8b7b058eadbe679e6c9b5e2e",
  "gitTreeState": "clean",
  "buildDate": "2023-03-20T21:30:46Z",
  "goVersion": "go1.19.7",
  "compiler": "gc",
  "platform": "linux/amd64"
}

# eks nodegroup 조회 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# eksctl get nodegroup --cluster $CLUSTER_NAME --name $CLUSTER_NAME-nodegroup
CLUSTER	NODEGROUP		STATUS	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID	ASG NAME							TYPE
eks-ami	eks-ami-nodegroup	ACTIVE	2023-04-29T11:39:57Z	2		2		2			t3.medium	AL2_x86_64	eks-eks-ami-nodegroup-00c3e667-fc25-5d32-cda1-5e45fcab08c5	managed
(yjkim@eks-ami:N/A) [root@myeks-host ~]# aws eks describe-nodegroup --cluster-name $CLUSTER_NAME --nodegroup-name $CLUSTER_NAME-nodegroup | jq
{
  "nodegroup": {
    "nodegroupName": "eks-ami-nodegroup",
    "nodegroupArn": "arn:aws:eks:ap-northeast-3:493327745939:nodegroup/eks-ami/eks-ami-nodegroup/00c3e667-fc25-5d32-cda1-5e45fcab08c5",
    "clusterName": "eks-ami",
    "version": "1.24",
    "releaseVersion": "1.24.11-20230411",
    "createdAt": "2023-04-29T20:39:57.681000+09:00",
    "modifiedAt": "2023-04-29T22:00:15.872000+09:00",
    "status": "ACTIVE",
    "capacityType": "ON_DEMAND",
    "scalingConfig": {
      "minSize": 2,
      "maxSize": 2,
      "desiredSize": 2
    },
    "instanceTypes": [
      "t3.medium"
    ],
    "subnets": [
      "subnet-01497e12118abcbe2",
      "subnet-0ef48df128ded06ad"
    ],
    "amiType": "AL2_x86_64",
    "nodeRole": "arn:aws:iam::493327745939:role/eksctl-eks-ami-nodegroup-eks-ami-NodeInstanceRole-4AQDRMY14B3F",
    "labels": {
      "alpha.eksctl.io/nodegroup-name": "eks-ami-nodegroup",
      "alpha.eksctl.io/cluster-name": "eks-ami"
    },
    "resources": {
      "autoScalingGroups": [
        {
          "name": "eks-eks-ami-nodegroup-00c3e667-fc25-5d32-cda1-5e45fcab08c5"
        }
      ]
    },
    "health": {
      "issues": []
    },
    "updateConfig": {
      "maxUnavailable": 1
    },
    "launchTemplate": {
      "name": "eksctl-eks-ami-nodegroup-eks-ami-nodegroup",
      "version": "1",
      "id": "lt-00adb52f863d080fb"
    },
    "tags": {
      "aws:cloudformation:stack-name": "eksctl-eks-ami-nodegroup-eks-ami-nodegroup",
      "alpha.eksctl.io/cluster-name": "eks-ami",
      "alpha.eksctl.io/nodegroup-name": "eks-ami-nodegroup",
      "aws:cloudformation:stack-id": "arn:aws:cloudformation:ap-northeast-3:493327745939:stack/eksctl-eks-ami-nodegroup-eks-ami-nodegroup/7f09a970-e682-11ed-9866-060ff34f3090",
      "eksctl.cluster.k8s.io/v1alpha1/cluster-name": "eks-ami",
      "aws:cloudformation:logical-id": "ManagedNodeGroup",
      "alpha.eksctl.io/nodegroup-type": "managed",
      "alpha.eksctl.io/eksctl-version": "0.139.0"
    }
  }
}

# node 정보 조회 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
NAME                                               STATUS   ROLES    AGE   VERSION                INSTANCE-TYPE   CAPACITYTYPE   ZONE
ip-192-168-1-69.ap-northeast-3.compute.internal    Ready    <none>   80m   v1.24.11-eks-a59e1f0   t3.medium       ON_DEMAND      ap-northeast-3a
ip-192-168-2-103.ap-northeast-3.compute.internal   Ready    <none>   80m   v1.24.11-eks-a59e1f0   t3.medium       ON_DEMAND      ap-northeast-3c
(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get node --label-columns=node.kubernetes.io/instance-type
NAME                                               STATUS   ROLES    AGE   VERSION                INSTANCE-TYPE
ip-192-168-1-69.ap-northeast-3.compute.internal    Ready    <none>   81m   v1.24.11-eks-a59e1f0   t3.medium
ip-192-168-2-103.ap-northeast-3.compute.internal   Ready    <none>   81m   v1.24.11-eks-a59e1f0   t3.medium
(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get node -owide
NAME                                               STATUS   ROLES    AGE   VERSION                INTERNAL-IP     EXTERNAL-IP     OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-1-69.ap-northeast-3.compute.internal    Ready    <none>   81m   v1.24.11-eks-a59e1f0   192.168.1.69    13.208.193.18   Amazon Linux 2   5.10.176-157.645.amzn2.x86_64   containerd://1.6.19
ip-192-168-2-103.ap-northeast-3.compute.internal   Ready    <none>   81m   v1.24.11-eks-a59e1f0   192.168.2.103   15.152.80.239   Amazon Linux 2   5.10.176-157.645.amzn2.x86_64   containerd://1.6.19

# pod 의 ip 확인 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get po -A -o wide
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
default       nginx                      1/1     Running   0          9s    192.168.1.238   ip-192-168-1-69.ap-northeast-3.compute.internal    <none>           <none>
kube-system   aws-node-frkjq             1/1     Running   0          82m   192.168.1.69    ip-192-168-1-69.ap-northeast-3.compute.internal    <none>           <none>
kube-system   aws-node-xld4d             1/1     Running   0          82m   192.168.2.103   ip-192-168-2-103.ap-northeast-3.compute.internal   <none>           <none>
kube-system   coredns-5bc54f57fc-6jv89   1/1     Running   0          89m   192.168.2.202   ip-192-168-2-103.ap-northeast-3.compute.internal   <none>           <none>
kube-system   coredns-5bc54f57fc-skgkl   1/1     Running   0          89m   192.168.2.110   ip-192-168-2-103.ap-northeast-3.compute.internal   <none>           <none>
kube-system   kube-proxy-8krnl           1/1     Running   0          82m   192.168.2.103   ip-192-168-2-103.ap-northeast-3.compute.internal   <none>           <none>
kube-system   kube-proxy-z8sld           1/1     Running   0          82m   192.168.1.69    ip-192-168-1-69.ap-northeast-3.compute.internal    <none>           <none>

# image 정보 확인 
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq -c
      2 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni:v1.11.4-eksbuild.1
      2 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/coredns:v1.8.7-eksbuild.3
      2 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/kube-proxy:v1.24.7-minimal-eksbuild.2
      1 nginx

```

### 노드 네트워크 정보 확인 

```sh
 aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
---------------------------------------------------------------------------------
|                               DescribeInstances                               |
+---------------------------------+----------------+----------------+-----------+
|          InstanceName           | PrivateIPAdd   |  PublicIPAdd   |  Status   |
+---------------------------------+----------------+----------------+-----------+
|  eks-ami-eks-ami-nodegroup-Node |  192.168.2.103 |  15.152.80.239 |  running  |
|  myeks-host                     |  192.168.1.100 |  13.208.242.15 |  running  |
|  eks-ami-eks-ami-nodegroup-Node |  192.168.1.69  |  13.208.193.18 |  running  |
+---------------------------------+----------------+----------------+-----------+

N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3c -o jsonpath={.items[0].status.addresses[0].address})
echo $N1, $N2
192.168.1.69, 192.168.2.103

ping -c 2 $N1
ping -c 2 $N2

# 노드 보안그룹 ID 확인
aws ec2 describe-security-groups --filters Name=group-name,Values=*nodegroup* --query "SecurityGroups[*].[GroupId]" --output text
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*nodegroup* --query "SecurityGroups[*].[GroupId]" --output text)
echo $NGSGID
sg-0f84597b8a339c05e

# 노드 보안그룹에 규칙 추가 
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0620272902a6e4f05",
            "GroupId": "sg-0f84597b8a339c05e",
            "GroupOwnerId": "493327745939",
            "IsEgress": false,
            "IpProtocol": "-1",
            "FromPort": -1,
            "ToPort": -1,
            "CidrIpv4": "192.168.1.100/32"
        }
    ]
}

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ping -c 2 $N1
PING 192.168.1.69 (192.168.1.69) 56(84) bytes of data.
64 bytes from 192.168.1.69: icmp_seq=1 ttl=255 time=1.09 ms
^C
--- 192.168.1.69 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.099/1.099/1.099/0.000 ms
(yjkim@eks-ami:N/A) [root@myeks-host ~]# ping -c 2 $N2
PING 192.168.2.103 (192.168.2.103) 56(84) bytes of data.
64 bytes from 192.168.2.103: icmp_seq=1 ttl=255 time=0.810 ms
^C
--- 192.168.2.103 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.810/0.810/0.810/0.000 ms

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 hostname
ip-192-168-1-69.ap-northeast-3.compute.internal

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N2 hostname
ip-192-168-2-103.ap-northeast-3.compute.internal

# AWS VPC CNI 사용 확인, 조회 되는게 없음 
kubectl -n kube-system get ds aws-node
kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2

# 파드 IP 확인
kubectl get pod -n kube-system -o wide
kubectl get pod -n kube-system -l k8s-app=kube-dns -owide

ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ip -c addr
ssh -i ~/.ssh/id_rsa ec2-user@$N2 sudo ip -c addr
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ip -c route
ssh -i ~/.ssh/id_rsa ec2-user@$N2 sudo ip -c route
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo iptables -t nat -S
ssh -i ~/.ssh/id_rsa ec2-user@$N2 sudo iptables -t nat -S

# 위의 eks nodegroup 의 private ip 를 가지고 있음 
$ ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ip -c addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:0d:a6:fe:e3:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.69/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 2939sec preferred_lft 2939sec
    inet6 fe80::c0d:a6ff:fefe:e352/64 scope link
       valid_lft forever preferred_lft forever
3: enic440f455693@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether fe:46:16:d7:4e:14 brd ff:ff:ff:ff:ff:ff link-netns cni-710508d0-9565-a5a5-4303-137f2e2cac32
    inet6 fe80::fc46:16ff:fed7:4e14/64 scope link
       valid_lft forever preferred_lft forever
4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:89:88:d1:bc:8c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.76/24 brd 192.168.1.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::c89:88ff:fed1:bc8c/64 scope link
       valid_lft forever preferred_lft forever

$ sh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ip -c route
default via 192.168.1.1 dev eth0
169.254.169.254 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.69
192.168.1.238 dev enic440f455693 scope link

$ ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N AWS-CONNMARK-CHAIN-0
-N AWS-CONNMARK-CHAIN-1
-N AWS-SNAT-CHAIN-0
-N AWS-SNAT-CHAIN-1
-N KUBE-KUBELET-CANARY
-N KUBE-MARK-DROP
-N KUBE-MARK-MASQ
-N KUBE-NODEPORTS
-N KUBE-POSTROUTING
-N KUBE-PROXY-CANARY
-N KUBE-SEP-FPLWHGKOZYTIXLBA
-N KUBE-SEP-IUXXGU6PHFM4XOIR
-N KUBE-SEP-OG7EYTSQVWNOPM2B
-N KUBE-SEP-QITN6PL3Q2CB3WD5
-N KUBE-SEP-QOWY5WNKXGKKZOZD
-N KUBE-SEP-ZMSJI3VRIGJXZA3J
-N KUBE-SERVICES
-N KUBE-SVC-ERIFXISQEP7F7OF4
-N KUBE-SVC-NPX46M4PTMTKRN6Y
-N KUBE-SVC-TCOU7JCQXEZGVUNU
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -i eni+ -m comment --comment "AWS, outbound connections" -m state --state NEW -j AWS-CONNMARK-CHAIN-0
-A PREROUTING -m comment --comment "AWS, CONNMARK" -j CONNMARK --restore-mark --nfmask 0x80 --ctmask 0x80
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -m comment --comment "AWS SNAT CHAIN" -j AWS-SNAT-CHAIN-0
-A AWS-CONNMARK-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS CONNMARK CHAIN, VPC CIDR" -j AWS-CONNMARK-CHAIN-1
-A AWS-CONNMARK-CHAIN-1 -m comment --comment "AWS, CONNMARK" -j CONNMARK --set-xmark 0x80/0x80
-A AWS-SNAT-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS SNAT CHAIN" -j AWS-SNAT-CHAIN-1
-A AWS-SNAT-CHAIN-1 ! -o vlan+ -m comment --comment "AWS, SNAT" -m addrtype ! --dst-type LOCAL -j SNAT --to-source 192.168.1.69 --random-fully
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
-A KUBE-SEP-FPLWHGKOZYTIXLBA -s 192.168.2.110/32 -m comment --comment "kube-system/kube-dns:dns" -j KUBE-MARK-MASQ
-A KUBE-SEP-FPLWHGKOZYTIXLBA -p udp -m comment --comment "kube-system/kube-dns:dns" -m udp -j DNAT --to-destination 192.168.2.110:53
-A KUBE-SEP-IUXXGU6PHFM4XOIR -s 192.168.2.202/32 -m comment --comment "kube-system/kube-dns:dns" -j KUBE-MARK-MASQ
-A KUBE-SEP-IUXXGU6PHFM4XOIR -p udp -m comment --comment "kube-system/kube-dns:dns" -m udp -j DNAT --to-destination 192.168.2.202:53
-A KUBE-SEP-OG7EYTSQVWNOPM2B -s 192.168.2.110/32 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-MARK-MASQ
-A KUBE-SEP-OG7EYTSQVWNOPM2B -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp" -m tcp -j DNAT --to-destination 192.168.2.110:53
-A KUBE-SEP-QITN6PL3Q2CB3WD5 -s 192.168.1.84/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-QITN6PL3Q2CB3WD5 -p tcp -m comment --comment "default/kubernetes:https" -m tcp -j DNAT --to-destination 192.168.1.84:443
-A KUBE-SEP-QOWY5WNKXGKKZOZD -s 192.168.2.41/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-QOWY5WNKXGKKZOZD -p tcp -m comment --comment "default/kubernetes:https" -m tcp -j DNAT --to-destination 192.168.2.41:443
-A KUBE-SEP-ZMSJI3VRIGJXZA3J -s 192.168.2.202/32 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-MARK-MASQ
-A KUBE-SEP-ZMSJI3VRIGJXZA3J -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp" -m tcp -j DNAT --to-destination 192.168.2.202:53
-A KUBE-SERVICES -d 10.100.0.10/32 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp cluster IP" -m tcp --dport 53 -j KUBE-SVC-ERIFXISQEP7F7OF4
-A KUBE-SERVICES -d 10.100.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -d 10.100.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
-A KUBE-SVC-ERIFXISQEP7F7OF4 -m comment --comment "kube-system/kube-dns:dns-tcp -> 192.168.2.110:53" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-OG7EYTSQVWNOPM2B
-A KUBE-SVC-ERIFXISQEP7F7OF4 -m comment --comment "kube-system/kube-dns:dns-tcp -> 192.168.2.202:53" -j KUBE-SEP-ZMSJI3VRIGJXZA3J
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https -> 192.168.1.84:443" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-QITN6PL3Q2CB3WD5
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https -> 192.168.2.41:443" -j KUBE-SEP-QOWY5WNKXGKKZOZD
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m comment --comment "kube-system/kube-dns:dns -> 192.168.2.110:53" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-FPLWHGKOZYTIXLBA
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m comment --comment "kube-system/kube-dns:dns -> 192.168.2.202:53" -j KUBE-SEP-IUXXGU6PHFM4XOIR

```

### 노드 프로세스 정보 확인 

```sh 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubelet-args.conf, 30-kubelet-extra-args.conf
   Active: active (running) since Sat 2023-04-29 11:41:14 UTC; 1h 33min ago
     Docs: https://github.com/kubernetes/kubernetes
  Process: 2830 ExecStartPre=/sbin/iptables -P FORWARD ACCEPT -w 5 (code=exited, status=0/SUCCESS)
 Main PID: 2840 (kubelet)
    Tasks: 15
   Memory: 79.6M
   CGroup: /runtime.slice/kubelet.service
           └─2840 /usr/bin/kubelet --config /etc/kubernetes/kubelet/kubelet-config.json --kubeconfig /var/lib/kubelet/kubeconfig --container-runtime-endpoint unix:///run/containerd/containerd.sock --image-credential-provider-config /etc/eks/ecr-credential-provider/ecr-credential-provider-config --image-credential-provider-bin-dir /etc/eks/ecr-credential-provider --node-ip=192.168.1.69 --pod-infra-container-image=602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.5 --v=2 --cloud-provider=aws --container-runtime=remote --node-labels=eks.amazonaws.com/sourceLaunchTemplateVersion=1,alpha.eksctl.io/cluster-name=eks-ami,alpha.eksctl.io/nodegroup-name=eks-ami-nodegroup,eks.amazonaws.com/nodegroup-image=ami-0fd395328f1733380,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=eks-ami-nodegroup,eks.amazonaws.com/sourceLaunchTemplateId=lt-00adb52f863d080fb --max-pods=17

Apr 29 13:03:31 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:31.985655    2840 kubelet.go:2093] "SyncLoop ADD" source="api" pods=[default/nginx]
Apr 29 13:03:31 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:31.985696    2840 topology_manager.go:200] "Topology Admit Handler"
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.146919    2840 reconciler.go:352] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-4fmrp\" (UniqueName: \"kubernetes.io/projected/86c65bdf-b6fa-4d56-ba35-dec83dcf6450-kube-api-access-4fmrp\") pod \"nginx\" (UID: \"86c65bdf-b6fa-4d56-ba35-dec83dcf6450\") " pod="default/nginx"
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.247219    2840 reconciler.go:264] "operationExecutor.MountVolume started for volume \"kube-api-access-4fmrp\" (UniqueName: \"kubernetes.io/projected/86c65bdf-b6fa-4d56-ba35-dec83dcf6450-kube-api-access-4fmrp\") pod \"nginx\" (UID: \"86c65bdf-b6fa-4d56-ba35-dec83dcf6450\") " pod="default/nginx"
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.260382    2840 operation_generator.go:703] "MountVolume.SetUp succeeded for volume \"kube-api-access-4fmrp\" (UniqueName: \"kubernetes.io/projected/86c65bdf-b6fa-4d56-ba35-dec83dcf6450-kube-api-access-4fmrp\") pod \"nginx\" (UID: \"86c65bdf-b6fa-4d56-ba35-dec83dcf6450\") " pod="default/nginx"
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.318103    2840 kuberuntime_manager.go:469] "No sandbox for pod can be found. Need to start a new one" pod="default/nginx"
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.696925    2840 provider.go:102] Refreshing cache for provider: *credentialprovider.defaultDockerConfigProvider
Apr 29 13:03:32 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:32.697063    2840 provider.go:82] Docker config file not found: couldn't find valid .dockercfg after checking in [/var/lib/kubelet   /]
Apr 29 13:03:33 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:33.114877    2840 kubelet.go:2131] "SyncLoop (PLEG): event for pod" pod="default/nginx" event=&{ID:86c65bdf-b6fa-4d56-ba35-dec83dcf6450 Type:ContainerStarted Data:dec6b2f22c29eeae136cdcd16f681c806477b8dcfd311aedbdd3c7e00fd88a7c}
Apr 29 13:03:39 ip-192-168-1-69.ap-northeast-3.compute.internal kubelet[2840]: I0429 13:03:39.133185    2840 kubelet.go:2131] "SyncLoop (PLEG): event for pod" pod="default/nginx" event=&{ID:86c65bdf-b6fa-4d56-ba35-dec83dcf6450 Type:ContainerStarted Data:6359188b5c8da88ae5c67cfdf07d527dd20f43614e76343a90cc83f766eb7be8}
(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo pstree
systemd-+-2*[agetty]
        |-amazon-ssm-agen-+-ssm-agent-worke---8*[{ssm-agent-worke}]
        |                 `-8*[{amazon-ssm-agen}]
        |-anacron
        |-auditd---{auditd}
        |-chronyd
        |-containerd---16*[{containerd}]
        |-containerd-shim-+-nginx---2*[nginx]
        |                 |-pause
        |                 `-10*[{containerd-shim}]
        |-containerd-shim-+-kube-proxy---5*[{kube-proxy}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-bash-+-aws-k8s-agent---8*[{aws-k8s-agent}]
        |                 |      `-tee
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-crond
        |-dbus-daemon
        |-2*[dhclient]
        |-gssproxy---5*[{gssproxy}]
        |-irqbalance---{irqbalance}
        |-kubelet---14*[{kubelet}]
        |-lvmetad
        |-master-+-pickup
        |        `-qmgr
        |-rngd
        |-rpcbind
        |-rsyslogd---2*[{rsyslogd}]
        |-sshd-+-2*[sshd---sshd]
        |      |-sshd
        |      `-sshd---sshd---sudo---pstree
        |-systemd-journal
        |-systemd-logind
        `-systemd-udevd

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ps afxuwww
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         2  0.0  0.0      0     0 ?        S    11:40   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [rcu_gp]
root         4  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [rcu_par_gp]
root         6  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kworker/0:0H-ev]
root         8  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [mm_percpu_wq]
root         9  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [rcu_tasks_rude_]
root        10  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [rcu_tasks_trace]
root        11  0.0  0.0      0     0 ?        R    11:40   0:00  \_ [ksoftirqd/0]
root        12  0.0  0.0      0     0 ?        I    11:40   0:00  \_ [rcu_sched]
root        13  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [migration/0]
root        14  0.0  0.0      0     0 ?        I    11:40   0:00  \_ [kworker/0:1-cgr]
root        15  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [cpuhp/0]
root        16  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [cpuhp/1]
root        17  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [migration/1]
root        18  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [ksoftirqd/1]
root        19  0.0  0.0      0     0 ?        I    11:40   0:00  \_ [kworker/1:0-cgr]
root        20  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kworker/1:0H-ev]
root        23  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [kdevtmpfs]
root        24  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [netns]
root        26  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [kauditd]
root       305  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [khungtaskd]
root       306  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [oom_reaper]
root       307  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [writeback]
root       308  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [kcompactd0]
root       310  0.0  0.0      0     0 ?        SN   11:40   0:00  \_ [ksmd]
root       311  0.0  0.0      0     0 ?        SN   11:40   0:00  \_ [khugepaged]
root       366  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kintegrityd]
root       367  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kblockd]
root       369  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [blkcg_punt_bio]
root       479  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [tpm_dev_wq]
root       485  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [md]
root       490  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [edac-poller]
root       495  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [watchdogd]
root       586  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kworker/1:1H-xf]
root       626  0.0  0.0      0     0 ?        S    11:40   0:00  \_ [kswapd0]
root       628  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfsalloc]
root       629  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs_mru_cache]
root       632  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kthrotld]
root       674  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [nvme-wq]
root       676  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [nvme-reset-wq]
root       678  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [nvme-delete-wq]
root       710  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [ipv6_addrconf]
root       719  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kworker/0:1H-kb]
root       723  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kstrp]
root       733  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [zswap-shrink]
root       734  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [kworker/u5:0]
root      1176  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-buf/nvme0n1]
root      1177  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-conv/nvme0n]
root      1178  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-cil/nvme0n1]
root      1179  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-reclaim/nvm]
root      1180  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-eofblocks/n]
root      1181  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xfs-log/nvme0n1]
root      1182  0.0  0.0      0     0 ?        S    11:40   0:01  \_ [xfsaild/nvme0n1]
root      1656  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [ena]
root      1709  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [cryptd]
root      1804  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [rpciod]
root      1805  0.0  0.0      0     0 ?        I<   11:40   0:00  \_ [xprtiod]
root      2834  0.0  0.0   4228   736 ?        S    11:41   0:00  \_ bpfilter_umh
root      7487  0.0  0.0      0     0 ?        I    11:55   0:00  \_ [kworker/1:1-cgr]
root     11333  0.0  0.0      0     0 ?        I    12:09   0:00  \_ [kworker/u4:1-xf]
root     32197  0.0  0.0      0     0 ?        I    12:56   0:00  \_ [kworker/0:0-cgr]
root      3989  0.0  0.0      0     0 ?        I    13:09   0:00  \_ [kworker/u4:2-ev]
root      3992  0.0  0.0      0     0 ?        I    13:09   0:00  \_ [kworker/0:2-mm_]
root      3993  0.0  0.0      0     0 ?        I    13:09   0:00  \_ [kworker/1:2-eve]
root      4613  0.0  0.0      0     0 ?        I    13:11   0:00  \_ [kworker/1:3-eve]
root      4657  0.0  0.0      0     0 ?        I    13:12   0:00  \_ [kworker/0:3-cgr]
root         1  0.0  0.1 123668  5656 ?        Ss   11:40   0:03 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root      1245  0.0  0.2  48320  7956 ?        Ss   11:40   0:00 /usr/lib/systemd/systemd-journald
root      1270  0.0  0.0 116756  2144 ?        Ss   11:40   0:00 /usr/sbin/lvmetad -f
root      1650  0.0  0.1  42824  4008 ?        Ss   11:40   0:00 /usr/lib/systemd/systemd-udevd
root      1790  0.0  0.0  57664  1904 ?        S<sl 11:40   0:00 /sbin/auditd
rpc       1819  0.0  0.0  67280  3300 ?        Ss   11:41   0:00 /sbin/rpcbind -w
root      1821  0.0  0.0 101992  1560 ?        Ssl  11:41   0:00 /usr/sbin/irqbalance --foreground
root      1822  0.0  0.0  26396  2956 ?        Ss   11:41   0:00 /usr/lib/systemd/systemd-logind
dbus      1823  0.0  0.1  56316  4124 ?        Ss   11:41   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
rngd      1848  0.0  0.1  94144  4656 ?        Ss   11:41   0:00 /sbin/rngd -f --fill-watermark=0 --exclude=jitter
chrony    1851  0.0  0.0 118272  3768 ?        S    11:41   0:00 /usr/sbin/chronyd -F 2
root      1858  0.0  0.0 212004  3216 ?        Ssl  11:41   0:00 /usr/sbin/gssproxy -D
root      2065  0.0  0.1  98688  4540 ?        Ss   11:41   0:00 /sbin/dhclient -q -lf /var/lib/dhclient/dhclient--eth0.lease -pf /var/run/dhclient-eth0.pid eth0
root      2100  0.0  0.1  98688  4112 ?        Ss   11:41   0:00 /sbin/dhclient -6 -nw -lf /var/lib/dhclient/dhclient6--eth0.lease -pf /var/run/dhclient6-eth0.pid eth0
root      2261  0.0  0.1  88232  4720 ?        Ss   11:41   0:00 /usr/libexec/postfix/master -w
postfix   2262  0.0  0.1  88316  6720 ?        S    11:41   0:00  \_ pickup -l -t unix -u
postfix   2263  0.0  0.1  88388  6788 ?        S    11:41   0:00  \_ qmgr -l -t unix -u
root      2350  0.0  0.1 224728  4360 ?        Ssl  11:41   0:00 /usr/sbin/rsyslogd -n
root      2352  0.0  0.3 714420 13268 ?        Ssl  11:41   0:00 /usr/bin/amazon-ssm-agent
root      2450  0.0  0.6 724360 24888 ?        Sl   11:41   0:02  \_ /usr/bin/ssm-agent-worker
root      2359  0.0  0.0 133008  3120 ?        Ss   11:41   0:00 /usr/sbin/crond -n
root      2360  0.0  0.0 116812  1888 ttyS0    Ss+  11:41   0:00 /sbin/agetty --keep-baud 115200,38400,9600 ttyS0 vt220
root      2361  0.0  0.0 117164  1548 tty1     Ss+  11:41   0:00 /sbin/agetty --noclear tty1 linux
root      2409  0.0  0.1 108772  7656 ?        Ss   11:41   0:00 /usr/sbin/sshd -D
root      5780  0.0  0.2 148444  8500 ?        Ss   13:15   0:00  \_ sshd: unknown [priv]
sshd      5828  0.0  0.1 113196  6664 ?        S    13:15   0:00  |   \_ sshd: unknown [net]
root      5803  0.0  0.1 108772  7116 ?        Ss   13:15   0:00  \_ sshd: [accepted]
sshd      5918  0.0  0.1 113108  5128 ?        S    13:15   0:00  |   \_ sshd: [net]
root      5941  0.0  0.2 146444  8436 ?        Ss   13:15   0:00  \_ sshd: ec2-user [priv]
ec2-user  5973  0.0  0.0 146444  3428 ?        S    13:15   0:00      \_ sshd: ec2-user@notty
root      5974  0.0  0.1 237736  7032 ?        Ss   13:15   0:00          \_ sudo ps afxuwww
root      5981  0.0  0.0 160380  3916 ?        R    13:15   0:00              \_ ps afxuwww
root      2726  0.5  1.5 1924728 60584 ?       Ssl  11:41   0:31 /usr/bin/containerd
root      2840  0.9  2.5 1905436 98836 ?       Ssl  11:41   0:56 /usr/bin/kubelet --config /etc/kubernetes/kubelet/kubelet-config.json --kubeconfig /var/lib/kubelet/kubeconfig --container-runtime-endpoint unix:///run/containerd/containerd.sock --image-credential-provider-config /etc/eks/ecr-credential-provider/ecr-credential-provider-config --image-credential-provider-bin-dir /etc/eks/ecr-credential-provider --node-ip=192.168.1.69 --pod-infra-container-image=602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.5 --v=2 --cloud-provider=aws --container-runtime=remote --node-labels=eks.amazonaws.com/sourceLaunchTemplateVersion=1,alpha.eksctl.io/cluster-name=eks-ami,alpha.eksctl.io/nodegroup-name=eks-ami-nodegroup,eks.amazonaws.com/nodegroup-image=ami-0fd395328f1733380,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=eks-ami-nodegroup,eks.amazonaws.com/sourceLaunchTemplateId=lt-00adb52f863d080fb --max-pods=17
root      2974  0.0  0.2 712736 10868 ?        Sl   11:41   0:01 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id ccfafe70490cbd13867e30b9fa1ccb8f2215c12cee425c11be8d481bfb426093 -address /run/containerd/containerd.sock
65535     3019  0.0  0.0    972     4 ?        Ss   11:41   0:00  \_ /pause
root      3099  0.0  0.9 745904 37116 ?        Ssl  11:41   0:00  \_ kube-proxy --v=2 --config=/var/lib/kube-proxy-config/config --hostname-override=ip-192-168-1-69.ap-northeast-3.compute.internal
root      2975  0.1  0.2 712736 10988 ?        Sl   11:41   0:06 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 1874959e34e4514bbfe10da3acdaac3e7830bfc54f1f4bda85f5484a522ef43a -address /run/containerd/containerd.sock
65535     3017  0.0  0.0    972     4 ?        Ss   11:41   0:00  \_ /pause
root      3336  0.0  0.0  11568  2688 ?        Ss   11:41   0:00  \_ bash /app/entrypoint.sh
root      3353  0.0  1.2 753532 47392 ?        Sl   11:41   0:02      \_ ./aws-k8s-agent
root      3354  0.0  0.0   4248   768 ?        S    11:41   0:00      \_ tee -i aws-k8s-agent.log
root      8919  0.0  0.0 132108  2500 ?        Ss   12:01   0:00 /usr/sbin/anacron -s
root      1934  0.0  0.2 712480 10400 ?        Sl   13:03   0:00 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id dec6b2f22c29eeae136cdcd16f681c806477b8dcfd311aedbdd3c7e00fd88a7c -address /run/containerd/containerd.sock
65535     1954  0.0  0.0    972     4 ?        Ss   13:03   0:00  \_ /pause
root      2025  0.0  0.1   8940  5592 ?        Ss   13:03   0:00  \_ nginx: master process nginx -g daemon off;
101       2059  0.0  0.0   9328  2524 ?        S    13:03   0:00      \_ nginx: worker process
101       2060  0.0  0.0   9328  2524 ?        S    13:03   0:00      \_ nginx: worker process

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ps axf |grep /usr/bin/containerd
 2726 ?        Ssl    0:31 /usr/bin/containerd
 2974 ?        Sl     0:01 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id ccfafe70490cbd13867e30b9fa1ccb8f2215c12cee425c11be8d481bfb426093 -address /run/containerd/containerd.sock
 2975 ?        Sl     0:06 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 1874959e34e4514bbfe10da3acdaac3e7830bfc54f1f4bda85f5484a522ef43a -address /run/containerd/containerd.sock
 1934 ?        Sl     0:00 /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id dec6b2f22c29eeae136cdcd16f681c806477b8dcfd311aedbdd3c7e00fd88a7c -address /run/containerd/containerd.sock

# 당연하게도 static pod 는 없다.
(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ls /etc/kubernetes/manifests/
```

### 노드 스토리지 정보 확인 

```sh 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
nvme0n1       259:0    0  30G  0 disk
├─nvme0n1p1   259:1    0  30G  0 part /
└─nvme0n1p128 259:2    0   1M  0 part

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 df -hT --type xfs
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/nvme0n1p1 xfs    30G  3.0G   28G  10% /
```


### mario deploy 

```sh
(yjkim@eks-ami:N/A) [root@myeks-host ~]# curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/1/mario.yaml

(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl apply -f mario.yaml
deployment.apps/mario created
service/mario created

(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get svc mario -o jsonpath={.status.loadBalancer.ingress[0].hostname} | awk '{ print "Maria URL = http://"$1 }'
Maria URL = http://a66666260528b4d2fa674da7fd3d1602-601292250.ap-northeast-3.elb.amazonaws.com
```

### 배포된 앱의 컨테이너 상태 확인 

```sh 
ssh -i ~/.ssh/id_rsa ec2-user@$N1 ctr --version
ssh -i ~/.ssh/id_rsa ec2-user@$N1 ctr 
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr ns list
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io container list
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io image list
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io task list
ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ps -c 3090

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr ns list
NAME   LABELS
k8s.io

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io container list
CONTAINER                                                           IMAGE                                                                                          RUNTIME
1874959e34e4514bbfe10da3acdaac3e7830bfc54f1f4bda85f5484a522ef43a    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/pause:3.5                                io.containerd.runc.v2
44af0efb127f3cbfeb7a873f9aea33d1a5ebf3e20f58fa6a9146c4cba29cf3fd    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/kube-proxy:v1.24.7-minimal-eksbuild.2    io.containerd.runc.v2
6359188b5c8da88ae5c67cfdf07d527dd20f43614e76343a90cc83f766eb7be8    docker.io/library/nginx:latest                                                                 io.containerd.runc.v2
6c9e43bd78d078e403a3f903f36f1037a15bd6da016dab29ec03a9c6c93cea1e    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/pause:3.5                                io.containerd.runc.v2
abf81564343be1d0eadade673b1a1c73196c391c85b6addadb8a55b91566e247    docker.io/pengbai/docker-supermario:latest                                                     io.containerd.runc.v2
ae4d43841bb357fe0a2d65ba8c864e5cf0b29d3281d5dde11d5a58de7c8e2373    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni-init:v1.11.4                  io.containerd.runc.v2
ccfafe70490cbd13867e30b9fa1ccb8f2215c12cee425c11be8d481bfb426093    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/pause:3.5                                io.containerd.runc.v2
d5d0e30f3ac2e8ec6a70a233b8a3c3565b3cac569426f50615e7629aa612a158    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni:v1.11.4                       io.containerd.runc.v2
dec6b2f22c29eeae136cdcd16f681c806477b8dcfd311aedbdd3c7e00fd88a7c    602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/pause:3.5                                io.containerd.runc.v2

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io image list | head -n 10
REF                                                                                                         TYPE                                                      DIGEST                                                                  SIZE      PLATFORMS                                                                                               LABELS
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni-init:v1.11.4                               application/vnd.docker.distribution.manifest.list.v2+json sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd 96.6 MiB  linux/amd64,linux/arm64/v8                                                                              io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni-init:v1.11.4-eksbuild.1                    application/vnd.docker.distribution.manifest.list.v2+json sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd 96.6 MiB  linux/amd64,linux/arm64/v8                                                                              io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni-init:v1.12.6                               application/vnd.docker.distribution.manifest.list.v2+json sha256:8d3a5343bd4ed9d297337eb3ddc218d6834b6ba3d6757af71324a0455114d002 21.2 MiB  linux/amd64,linux/arm64                                                                                 io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni-init:v1.12.6-eksbuild.1                    application/vnd.docker.distribution.manifest.list.v2+json sha256:8d3a5343bd4ed9d297337eb3ddc218d6834b6ba3d6757af71324a0455114d002 21.2 MiB  linux/amd64,linux/arm64                                                                                 io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni:v1.11.4                                    application/vnd.docker.distribution.manifest.list.v2+json sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4 97.0 MiB  linux/amd64,linux/arm64/v8                                                                              io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni:v1.11.4-eksbuild.1                         application/vnd.docker.distribution.manifest.list.v2+json sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4 97.0 MiB  linux/amd64,linux/arm64/v8                                                                              io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni:v1.12.6                                    application/vnd.docker.distribution.manifest.list.v2+json sha256:9e1e883fe1a246174df8433aa1ec9b00d44034938540b2e7c2981c0adc80ee2d 40.9 MiB  linux/amd64,linux/arm64                                                                                 io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/amazon-k8s-cni:v1.12.6-eksbuild.1                         application/vnd.docker.distribution.manifest.list.v2+json sha256:9e1e883fe1a246174df8433aa1ec9b00d44034938540b2e7c2981c0adc80ee2d 40.9 MiB  linux/amd64,linux/arm64                                                                                 io.cri-containerd.image=managed
602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/kube-proxy:v1.24.10-minimal-eksbuild.2                application/vnd.docker.distribution.manifest.list.v2+json sha256:577c4080f0b478d894c250dc56651dd7ff2881eb22141c9badac9e6458d91385 24.4 MiB  linux/amd64,linux/arm64                                                                                 io.cri-containerd.image=managed

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ctr -n k8s.io task list
TASK                                                                PID      STATUS
abf81564343be1d0eadade673b1a1c73196c391c85b6addadb8a55b91566e247    11438    RUNNING
1874959e34e4514bbfe10da3acdaac3e7830bfc54f1f4bda85f5484a522ef43a    3017     RUNNING
ccfafe70490cbd13867e30b9fa1ccb8f2215c12cee425c11be8d481bfb426093    3019     RUNNING
44af0efb127f3cbfeb7a873f9aea33d1a5ebf3e20f58fa6a9146c4cba29cf3fd    3099     RUNNING
d5d0e30f3ac2e8ec6a70a233b8a3c3565b3cac569426f50615e7629aa612a158    3336     RUNNING
dec6b2f22c29eeae136cdcd16f681c806477b8dcfd311aedbdd3c7e00fd88a7c    1954     RUNNING
6359188b5c8da88ae5c67cfdf07d527dd20f43614e76343a90cc83f766eb7be8    2025     RUNNING
6c9e43bd78d078e403a3f903f36f1037a15bd6da016dab29ec03a9c6c93cea1e    11282    RUNNING

(yjkim@eks-ami:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 sudo ps -c 2025
  PID CLS PRI TTY      STAT   TIME COMMAND
 2025 TS   19 ?        Ss     0:00 nginx: master process nginx -g daemon off;

```

### metrics 조회 

```sh 
(yjkim@eks-ami:N/A) [root@myeks-host ~]# kubectl get --raw /metrics | grep "etcd_db_total_size_in_bytes"
# HELP etcd_db_total_size_in_bytes [ALPHA] Total size of the etcd database file physically allocated in bytes.
# TYPE etcd_db_total_size_in_bytes gauge
etcd_db_total_size_in_bytes{endpoint="http://10.0.160.16:2379"} 2.609152e+06
etcd_db_total_size_in_bytes{endpoint="http://10.0.32.16:2379"} 2.654208e+06
etcd_db_total_size_in_bytes{endpoint="http://10.0.96.16:2379"} 2.613248e+06
```

### 생성한 eks cluster 삭제 

* eksctl delete cluster --name eks-ami

## bottlerocket 기반 nodegroup 로 배포하기 

다른 VM 의 게스트 OS 로 설치한다는게 쉽지는 않기 떄문에 amilinux2 vs bottlerocket 를 따로 준비 해야하지만 
많은 자료를 찾지는 않은 상태이며 eksctl 에서도 다른 os 를 지원하기 떄문에 위와 같이 설치 한 후에 비교 하는 그림으로 진행할 예정이다. 


```sh 
# https://eksctl.io/usage/schema/#nodeGroups-amiFamily
(yjkim@eks-ami:N/A) [root@myeks-host ~]# eksctl create cluster --help | grep ami
      --node-ami string                'auto-ssm', 'auto' or an AMI ID (advanced use)
      --node-ami-family string         'AmazonLinux2' for the Amazon EKS optimized AMI, or use 'Ubuntu2004' or 'Ubuntu1804' for the official Canonical EKS AMIs (default "AmazonLinux2")

CLUSTER_NAME=eks-bottlerocket

# debug
(N/A:N/A) [root@myeks-host ~]# eksctl create cluster --name $CLUSTER_NAME --region=$AWS_DEFAULT_REGION --nodegroup-name=$CLUSTER_NAME-nodegroup --node-type=t3.medium --node-volume-size=30 --vpc-public-subnets "$PubSubnet1,$PubSubnet2" --version 1.24  --node-ami-family  Bottlerocket --ssh-access --external-dns-access --dry-run | yh | grep  ami
  ipFamily: IPv4
- amiFamily: Bottlerocket

# 설치 
 eksctl create cluster --name $CLUSTER_NAME --region=$AWS_DEFAULT_REGION --nodegroup-name=$CLUSTER_NAME-nodegroup --node-type=t3.medium --node-volume-size=30 \
  --vpc-public-subnets "$PubSubnet1,$PubSubnet2" --version 1.24  --node-ami-family  Bottlerocket --ssh-access --external-dns-access --verbose 4
```


### 노드 네트워크 정보 확인 

```sh 
N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3c -o jsonpath={.items[0].status.addresses[0].address})
echo $N1, $N2


# SG 에 SSH 접속할 수 있는 인입정보 추가 
aws ec2 describe-security-groups --filters Name=group-name,Values=*nodegroup* --query "SecurityGroups[*].[GroupId]" --output text
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*nodegroup* --query "SecurityGroups[*].[GroupId]" --output text)
echo $NGSGID

# 노드 보안그룹에 eksctl-host 에서 노드(파드)에 접속 가능하게 룰(Rule) 추가 설정
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32

(yjkim@eks-bottlerocket:N/A) [root@myeks-host ~]# ping $N1
PING 192.168.1.120 (192.168.1.120) 56(84) bytes of data.
64 bytes from 192.168.1.120: icmp_seq=1 ttl=255 time=0.479 ms
64 bytes from 192.168.1.120: icmp_seq=2 ttl=255 time=0.169 ms


# 이제 접속 확인을.. ? hostname 혹은 그 외 도구 들이 없는것을 보임 
(yjkim@eks-bottlerocket:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1 hostname

bash: hostname: command not found
(yjkim@eks-bottlerocket:N/A) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$N1
          Welcome to Bottlerocket's admin container!
    ╱╲
   ╱┄┄╲   This container provides access to the Bottlerocket host
   │▗▖│   filesystems (see /.bottlerocket/rootfs) and contains common
  ╱│  │╲  tools for inspection and troubleshooting.  It is based on
  │╰╮╭╯│  Amazon Linux 2, and most things are in the same places you
    ╹╹    would find them on an AL2 host.

To permit more intrusive troubleshooting, including actions that mutate the
running state of the Bottlerocket host, we provide a tool called "sheltie"
(`sudo sheltie`).  When run, this tool drops you into a root shell in the
Bottlerocket host's root filesystem.
[ec2-user@admin]$ ip a
-bash: ip: command not found
[ec2-user@admin]$ ifconfig
-bash: ifconfig: command not found
[ec2-user@admin]$ ls -al
total 28

# sheltie 로 접속을 해야 보인다. 
[ec2-user@admin]$ sheltie
sheltie must be run as root, you can use 'sudo sheltie' in the admin container
[ec2-user@admin]$ sudo sheltie
bash-5.1# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:e7:7f:a6:80:94 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    altname ens5
    inet 192.168.1.120/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ce7:7fff:fea6:8094/64 scope link
       valid_lft forever preferred_lft forever

bash-5.1# ip -c addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:e7:7f:a6:80:94 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    altname ens5
    inet 192.168.1.120/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ce7:7fff:fea6:8094/64 scope link
       valid_lft forever preferred_lft forever

bash-5.1# ip -c route
default via 192.168.1.1 dev eth0 proto dhcp
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.120

bash-5.1# iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N AWS-CONNMARK-CHAIN-0
-N AWS-CONNMARK-CHAIN-1
-N AWS-SNAT-CHAIN-0
-N AWS-SNAT-CHAIN-1
-N KUBE-KUBELET-CANARY
-N KUBE-MARK-DROP
-N KUBE-MARK-MASQ
-N KUBE-NODEPORTS
-N KUBE-POSTROUTING
-N KUBE-PROXY-CANARY
-N KUBE-SEP-3G6QDH75A6S7AIA7
-N KUBE-SEP-AHW7CH2CAVYXJLNH
-N KUBE-SEP-CDHRXOESNUFTTBDW
-N KUBE-SEP-ETONYFORAJ5FNC5U
-N KUBE-SEP-TOVRX4LCNUAAPTLE
-N KUBE-SEP-ZG33WS3W6TJ2NNAM
-N KUBE-SERVICES
-N KUBE-SVC-ERIFXISQEP7F7OF4
-N KUBE-SVC-NPX46M4PTMTKRN6Y
-N KUBE-SVC-TCOU7JCQXEZGVUNU
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -i eni+ -m comment --comment "AWS, outbound connections" -m state --state NEW -j AWS-CONNMARK-CHAIN-0
-A PREROUTING -m comment --comment "AWS, CONNMARK" -j CONNMARK --restore-mark --nfmask 0x80 --ctmask 0x80
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -m comment --comment "AWS SNAT CHAIN" -j AWS-SNAT-CHAIN-0
-A AWS-CONNMARK-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS CONNMARK CHAIN, VPC CIDR" -j AWS-CONNMARK-CHAIN-1
-A AWS-CONNMARK-CHAIN-1 -m comment --comment "AWS, CONNMARK" -j CONNMARK --set-xmark 0x80/0x80
-A AWS-SNAT-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS SNAT CHAIN" -j AWS-SNAT-CHAIN-1
-A AWS-SNAT-CHAIN-1 ! -o vlan+ -m comment --comment "AWS, SNAT" -m addrtype ! --dst-type LOCAL -j SNAT --to-source 192.168.1.120 --random-fully
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
-A KUBE-SEP-3G6QDH75A6S7AIA7 -s 192.168.2.131/32 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-MARK-MASQ
-A KUBE-SEP-3G6QDH75A6S7AIA7 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp" -m tcp -j DNAT --to-destination 192.168.2.131:53
-A KUBE-SEP-AHW7CH2CAVYXJLNH -s 192.168.2.125/32 -m comment --comment "kube-system/kube-dns:dns" -j KUBE-MARK-MASQ
-A KUBE-SEP-AHW7CH2CAVYXJLNH -p udp -m comment --comment "kube-system/kube-dns:dns" -m udp -j DNAT --to-destination 192.168.2.125:53
-A KUBE-SEP-CDHRXOESNUFTTBDW -s 192.168.2.146/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-CDHRXOESNUFTTBDW -p tcp -m comment --comment "default/kubernetes:https" -m tcp -j DNAT --to-destination 192.168.2.146:443
-A KUBE-SEP-ETONYFORAJ5FNC5U -s 192.168.1.124/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-ETONYFORAJ5FNC5U -p tcp -m comment --comment "default/kubernetes:https" -m tcp -j DNAT --to-destination 192.168.1.124:443
-A KUBE-SEP-TOVRX4LCNUAAPTLE -s 192.168.2.131/32 -m comment --comment "kube-system/kube-dns:dns" -j KUBE-MARK-MASQ
-A KUBE-SEP-TOVRX4LCNUAAPTLE -p udp -m comment --comment "kube-system/kube-dns:dns" -m udp -j DNAT --to-destination 192.168.2.131:53
-A KUBE-SEP-ZG33WS3W6TJ2NNAM -s 192.168.2.125/32 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-MARK-MASQ
-A KUBE-SEP-ZG33WS3W6TJ2NNAM -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp" -m tcp -j DNAT --to-destination 192.168.2.125:53
-A KUBE-SERVICES -d 10.100.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SERVICES -d 10.100.0.10/32 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp cluster IP" -m tcp --dport 53 -j KUBE-SVC-ERIFXISQEP7F7OF4
-A KUBE-SERVICES -d 10.100.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
-A KUBE-SVC-ERIFXISQEP7F7OF4 -m comment --comment "kube-system/kube-dns:dns-tcp -> 192.168.2.125:53" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ZG33WS3W6TJ2NNAM
-A KUBE-SVC-ERIFXISQEP7F7OF4 -m comment --comment "kube-system/kube-dns:dns-tcp -> 192.168.2.131:53" -j KUBE-SEP-3G6QDH75A6S7AIA7
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https -> 192.168.1.124:443" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ETONYFORAJ5FNC5U
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https -> 192.168.2.146:443" -j KUBE-SEP-CDHRXOESNUFTTBDW
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m comment --comment "kube-system/kube-dns:dns -> 192.168.2.125:53" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-AHW7CH2CAVYXJLNH
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m comment --comment "kube-system/kube-dns:dns -> 192.168.2.131:53" -j KUBE-SEP-TOVRX4LCNUAAPTLE

```

### 노드 프로세스 정보 확인 

```sh 
# kubelet 정보는 잘 보인다.
bash-5.1# systemctl status kubelet
● kubelet.service - Kubelet
     Loaded: loaded (/x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/systemd/system/kubelet.service.d
             └─dockershim-symlink.conf
             /etc/systemd/system/kubelet.service.d
             └─exec-start.conf
             /x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/systemd/system/kubelet.service.d
             └─load-ipvs-modules.conf, make-kubelet-dirs.conf, prestart-pull-pause-ctr.conf
     Active: active (running) since Sat 2023-04-29 14:46:30 UTC; 26min ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 1128 ExecStartPre=/sbin/iptables -P FORWARD ACCEPT (code=exited, status=0/SUCCESS)
    Process: 1131 ExecStartPre=/bin/ln -sf /run/containerd/containerd.sock /run/dockershim.sock (code=exited, status=0/SUCCESS)
    Process: 1132 ExecStartPre=/usr/bin/mkdir -p /var/lib/kubelet/providers/secrets-store /var/lib/kubelet/node-feature-discovery/features.d (code=exited, status=0/SUCCESS)
    Process: 1133 ExecStartPre=/usr/bin/host-ctr --containerd-socket=/run/containerd/containerd.sock --namespace=k8s.io pull-image --source=${POD_INFRA_CONTAINER_IMAGE} --registry-config=/etc/host-containers/host-ctr.toml --skip-if-image-exists=true (code=exited, status=0/SUCCESS)
   Main PID: 1155 (kubelet)
      Tasks: 16 (limit: 4602)
     Memory: 193.1M
        CPU: 16.248s
     CGroup: /runtime.slice/kubelet.service
             └─ 1155 /usr/bin/kubelet --cloud-provider aws --kubeconfig /etc/kubernetes/kubelet/kubeconfig --config /etc/kubernetes/kubelet/config --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --containerd=/run/containerd/containerd.sock --root-dir /var/lib/kubelet --cert-dir /var/lib/kubelet/pki --node-ip 192.168.1.120 --node-labels alpha.eksctl.io/cluster-name=eks-bottlerocket,alpha.eksctl.io/nodegroup-name=eks-bottlerocket-nodegroup,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=eks-bottlerocket-nodegroup,eks.amazonaws.com/nodegroup-image=ami-06abf956961f9530b,eks.amazonaws.com/sourceLaunchTemplateId=lt-0b52cbe4a1361f8c6,eks.amazonaws.com/sourceLaunchTemplateVersion=1 --register-with-taints "" --pod-infra-container-image 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1

Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: I0429 14:46:31.524886    1155 reconciler.go:352] "operationExecutor.VerifyControllerAttachedVolume started for volume \"varlog\" (UniqueName: \"kubernetes.io/host-path/307f790a-dd46-4fb3-a8eb-1a834c75e222-varlog\") pod \"kube-proxy-5hh84\" (UID: \"307f790a-dd46-4fb3-a8eb-1a834c75e222\") " pod="kube-system/kube-proxy-5hh84"
Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: I0429 14:46:31.524952    1155 reconciler.go:352] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/307f790a-dd46-4fb3-a8eb-1a834c75e222-xtables-lock\") pod \"kube-proxy-5hh84\" (UID: \"307f790a-dd46-4fb3-a8eb-1a834c75e222\") " pod="kube-system/kube-proxy-5hh84"
Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: I0429 14:46:31.525009    1155 reconciler.go:352] "operationExecutor.VerifyControllerAttachedVolume started for volume \"config\" (UniqueName: \"kubernetes.io/configmap/307f790a-dd46-4fb3-a8eb-1a834c75e222-config\") pod \"kube-proxy-5hh84\" (UID: \"307f790a-dd46-4fb3-a8eb-1a834c75e222\") " pod="kube-system/kube-proxy-5hh84"
Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: I0429 14:46:31.525051    1155 reconciler.go:352] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-8kql7\" (UniqueName: \"kubernetes.io/projected/307f790a-dd46-4fb3-a8eb-1a834c75e222-kube-api-access-8kql7\") pod \"kube-proxy-5hh84\" (UID: \"307f790a-dd46-4fb3-a8eb-1a834c75e222\") " pod="kube-system/kube-proxy-5hh84"
Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: I0429 14:46:31.525068    1155 reconciler.go:169] "Reconciler: start to sync state"
Apr 29 14:46:31 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: E0429 14:46:31.560925    1155 watcher.go:152] Failed to watch directory "/sys/fs/cgroup/devices/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod307f790a_dd46_4fb3_a8eb_1a834c75e222.slice": inotify_add_watch /sys/fs/cgroup/devices/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod307f790a_dd46_4fb3_a8eb_1a834c75e222.slice: no such file or directory
Apr 29 14:46:37 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: E0429 14:46:37.222006    1155 kubelet.go:2352] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
Apr 29 14:46:42 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: E0429 14:46:42.247877    1155 kubelet.go:2352] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
Apr 29 14:46:47 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: E0429 14:46:47.248622    1155 kubelet.go:2352] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
Apr 29 14:46:52 ip-192-168-1-120.ap-northeast-3.compute.internal kubelet[1155]: E0429 14:46:52.250279    1155 kubelet.go:2352] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"

# pstree 는 명령어가 없다고 한다.

# 프로세스 정보도 잘 보인다. 
bash-5.1# ps afxuwww
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           2  0.0  0.0      0     0 ?        S    14:46   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [rcu_par_gp]
root           5  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [slub_flushwq]
root           6  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [netns]
root           8  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kworker/0:0H-events_highpri]
root          10  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [mm_percpu_wq]
root          11  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [rcu_tasks_rude_]
root          12  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [rcu_tasks_trace]
root          13  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [ksoftirqd/0]
root          14  0.0  0.0      0     0 ?        I    14:46   0:00  \_ [rcu_sched]
root          15  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [migration/0]
root          17  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [cpuhp/0]
root          18  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [cpuhp/1]
root          19  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [migration/1]
root          20  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [ksoftirqd/1]
root          21  0.0  0.0      0     0 ?        I    14:46   0:00  \_ [kworker/1:0-mm_percpu_wq]
root          22  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kworker/1:0H-events_highpri]
root          25  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [kdevtmpfs]
root          26  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [inet_frag_wq]
root          27  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [kauditd]
root          28  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [khungtaskd]
root          29  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [oom_reaper]
root          30  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [writeback]
root          31  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [kcompactd0]
root          32  0.0  0.0      0     0 ?        SN   14:46   0:00  \_ [khugepaged]
root          88  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kintegrityd]
root          89  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kblockd]
root          90  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [blkcg_punt_bio]
root          91  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [tpm_dev_wq]
root          92  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ata_sff]
root          93  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [md]
root          94  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [edac-poller]
root          95  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [watchdogd]
root         100  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kworker/0:1H-kblockd]
root         114  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [kswapd0]
root         130  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [xfsalloc]
root         132  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [xfs_mru_cache]
root         135  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kthrotld]
root         180  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [nvme-wq]
root         184  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [nvme-reset-wq]
root         186  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [nvme-delete-wq]
root         195  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [dm_bufio_cache]
root         198  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [mld]
root         199  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ipv6_addrconf]
root         226  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kstrp]
root         236  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [zswap-shrink]
root         258  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kdmflush]
root         259  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kverityd]
root         263  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kworker/u5:0]
root         264  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ext4-rsv-conver]
root         271  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [kworker/1:1H-kblockd]
root         738  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ena]
root         744  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [cryptd]
root         845  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [jbd2/nvme1n1p1-]
root         846  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ext4-rsv-conver]
root         885  0.0  0.0      0     0 ?        S    14:46   0:00  \_ [jbd2/nvme0n1p12]
root         887  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ext4-rsv-conver]
root         922  0.0  0.0      0     0 ?        I    14:46   0:00  \_ [kworker/u4:6-kverityd]
root         959  0.0  0.0      0     0 ?        I<   14:46   0:00  \_ [ext4-rsv-conver]
root        1606  0.0  0.0      0     0 ?        I    14:46   0:00  \_ [kworker/0:5-cgroup_destroy]
root        7561  0.0  0.0      0     0 ?        I    15:01   0:00  \_ [kworker/0:0-mm_percpu_wq]
root        7568  0.0  0.0      0     0 ?        I    15:01   0:00  \_ [kworker/1:1-events]
root        7668  0.0  0.0      0     0 ?        I    15:01   0:00  \_ [kworker/u4:0-events_unbound]
root       11341  0.0  0.0      0     0 ?        I    15:11   0:00  \_ [kworker/u4:1-kverityd]
root           1  0.1  0.2  88252 11172 ?        Ss   14:46   0:02 /sbin/init systemd.log_target=journal-or-kmsg systemd.log_color=0 systemd.show_status=true
root         289  0.0  0.1  25752  7560 ?        Ss   14:46   0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/systemd/systemd-journald
root         706  0.0  0.1  10596  6264 ?        Ss   14:46   0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/systemd/systemd-udevd
dbus         928  0.0  0.0   7312  3440 ?        Ss   14:46   0:00 /usr/bin/dbus-broker-launch --scope system
dbus         929  0.0  0.0   3992  2324 ?        S    14:46   0:00  \_ dbus-broker --log 4 --controller 9 --machine-id ec2fb3fdcf6995ba47539e8dbb372956 --max-bytes 536870912 --max-fds 4096 --max-matches 16384
root         931  0.0  0.0   2300    96 ?        Ss   14:46   0:00 /usr/sbin/acpid -c /usr/lib/acpid/events
root         937  0.0  0.1   7624  5552 ?        Ss   14:46   0:00 /usr/libexec/wicked/bin/wickedd-dhcp4 --systemd --foreground --log-level notice
root         938  0.0  0.1   7624  4856 ?        Ss   14:46   0:00 /usr/libexec/wicked/bin/wickedd-dhcp6 --systemd --foreground --log-level notice
root         958  0.0  0.2 146192  8864 ?        Ssl  14:46   0:00 /usr/bin/apiserver --datastore-path /var/lib/bottlerocket/datastore/current --socket-gid 274
root         960  0.0  0.1   7748  5456 ?        Ss   14:46   0:00 /usr/sbin/wickedd --systemd --foreground
root         964  0.0  0.1   7648  5360 ?        Ss   14:46   0:00 /usr/sbin/wickedd-nanny --systemd --foreground
chrony      1079  0.0  0.0  11720  2536 ?        Ss   14:46   0:00 /usr/sbin/chronyd -d -F -1
chrony      1081  0.0  0.0   3392   148 ?        S    14:46   0:00  \_ /usr/sbin/chronyd -d -F -1
root        1080  1.0  1.1 1505520 47124 ?       Ssl  14:46   0:17 /usr/bin/containerd --config /etc/host-containerd/config.toml
root        1101  1.0  1.4 1653496 57744 ?       Ssl  14:46   0:17 /usr/bin/containerd
root        1102  0.0  1.1 1356840 45444 ?       Ssl  14:46   0:00 /usr/bin/host-ctr run --container-id=admin --source=328549459982.dkr.ecr.ap-northeast-3.amazonaws.com/bottlerocket-admin:v0.10.0 --superpowered=true --registry-config=/etc/host-containers/host-ctr.toml
root        1103  0.0  1.1 1430316 44704 ?       Ssl  14:46   0:00 /usr/bin/host-ctr run --container-id=control --source=328549459982.dkr.ecr.ap-northeast-3.amazonaws.com/bottlerocket-control:v0.7.1 --superpowered=false --registry-config=/etc/host-containers/host-ctr.toml
root        1155  0.9  2.9 1955400 117532 ?      Ssl  14:46   0:16 /usr/bin/kubelet --cloud-provider aws --kubeconfig /etc/kubernetes/kubelet/kubeconfig --config /etc/kubernetes/kubelet/config --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --containerd=/run/containerd/containerd.sock --root-dir /var/lib/kubelet --cert-dir /var/lib/kubelet/pki --node-ip 192.168.1.120 --node-labels alpha.eksctl.io/cluster-name=eks-bottlerocket,alpha.eksctl.io/nodegroup-name=eks-bottlerocket-nodegroup,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=eks-bottlerocket-nodegroup,eks.amazonaws.com/nodegroup-image=ami-06abf956961f9530b,eks.amazonaws.com/sourceLaunchTemplateId=lt-0b52cbe4a1361f8c6,eks.amazonaws.com/sourceLaunchTemplateVersion=1 --register-with-taints  --pod-infra-container-image 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1
root        1272  0.1  0.4 1614040 16388 ?       Sl   14:46   0:01 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96 -address /run/containerd/containerd.sock
root        1337  0.0  0.0    972     4 ?        Ss   14:46   0:00  \_ /pause
root        1903  0.0  0.0  11568  2688 ?        Ss   14:46   0:00  \_ bash /app/entrypoint.sh
root        1931  0.0  1.2 753532 48260 ?        Sl   14:46   0:00      \_ ./aws-k8s-agent
root        1932  0.0  0.0   4248   816 ?        S    14:46   0:00      \_ tee -i aws-k8s-agent.log
root        1273  0.0  0.3 1466576 15056 ?       Sl   14:46   0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e -address /run/containerd/containerd.sock
root        1332  0.0  0.0    972     4 ?        Ss   14:46   0:00  \_ /pause
root        1397  0.0  0.9 745904 36984 ?        Ssl  14:46   0:00  \_ kube-proxy --v=2 --config=/var/lib/kube-proxy-config/config --hostname-override=ip-192-168-1-120.ap-northeast-3.compute.internal
root        1637  0.0  0.3 1540308 13136 ?       Sl   14:46   0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace default -id control -address /run/host-containerd/containerd.sock
root        1659  0.0  0.4 1307140 17088 ?       Ssl  14:46   0:00  \_ /usr/bin/amazon-ssm-agent
root        1867  0.0  0.6 1316416 25228 ?       Sl   14:46   0:00      \_ /usr/bin/ssm-agent-worker
root        1674  0.0  0.3 1540308 12488 ?       Sl   14:46   0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace default -id admin -address /run/host-containerd/containerd.sock
root        1693  0.0  0.1  41124  4796 ?        Ss   14:46   0:00  \_ /usr/lib/systemd/systemd --user --unit=admin.target
root        1864  0.0  0.1 108772  7704 ?        Ss   14:46   0:00      \_ /usr/sbin/sshd -e -D
root       10491  0.0  0.2 146440  8428 ?        Ss   15:09   0:00      |   \_ sshd: ec2-user [priv]
1000       10493  0.0  0.1 146440  4592 ?        S    15:09   0:00      |       \_ sshd: ec2-user@pts/0
1000       10494  0.0  0.0  13308  3272 ?        Ss   15:09   0:00      |           \_ -bash
root       11216  0.0  0.1 128924  7148 ?        S    15:10   0:00      |               \_ sudo sheltie
root       11217  0.0  0.0  16844  1048 ?        S    15:10   0:00      |                   \_ nsenter -t 1 -a /proc/11216/root/opt/bin/bash
root       11218  0.0  0.0   1528  1220 ?        S    15:10   0:00      |                       \_ /proc/11216/root/opt/bin/bash
root       12307  0.0  0.0   3280  2008 ?        R+   15:13   0:00      |                           \_ ps afxuwww
root        1865  0.0  0.0   4632   788 tty1     Ss+  14:46   0:00      \_ /sbin/agetty -o -- \u --noclear - xterm-256color
root        1866  0.0  0.0   4280   744 ttyS0    Ss+  14:46   0:00      \_ /sbin/agetty -o -- \u --keep-baud 115200,57600,38400,9600 - xterm-256color

## ps 정보에서 특이한 점이라고 하면 /usr/bin/host-ctr 이녀석 인듯 하다. 
## 잠깐 다른 블로그 등을 서치 해보면 control container, admin container 등을 제공 하고 있고 
## 관리 컨테이너는 shell, SSM 등을 실행 시킬 수 있으며 문서는 일부 제공 하고 있는듯 하다. : https://github.com/bottlerocket-os/bottlerocket-control-container/#connecting-to-aws-systems-manager-ssm
## admin 컨테이너는 ssh key 를 전달 한다던지 등을 할 수 있는것 으로 보인다. : https://github.com/bottlerocket-os/bottlerocket-admin-container
## 보안 등의 이슈로 shell, ssh 등이 구성 되지 않았다고 한다. https://rustrepo.com/repo/bottlerocket-os-bottlerocket#settings
## userdata 란에 특정 양식으로 전달하면 sysctl, kubernetes 구성 등을 전달 할 수 있다고 한다. https://github.com/bottlerocket-os/bottlerocket/blob/develop/README.md#description-of-settings

# runtime 는 containerd 로 보인다. 
bash-5.1# ps axf | grep containerd
   1080 ?        Ssl    0:17 /usr/bin/containerd --config /etc/host-containerd/config.toml
   1101 ?        Ssl    0:17 /usr/bin/containerd
   1155 ?        Ssl    0:16 /usr/bin/kubelet --cloud-provider aws --kubeconfig /etc/kubernetes/kubelet/kubeconfig --config /etc/kubernetes/kubelet/config --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --containerd=/run/containerd/containerd.sock --root-dir /var/lib/kubelet --cert-dir /var/lib/kubelet/pki --node-ip 192.168.1.120 --node-labels alpha.eksctl.io/cluster-name=eks-bottlerocket,alpha.eksctl.io/nodegroup-name=eks-bottlerocket-nodegroup,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=eks-bottlerocket-nodegroup,eks.amazonaws.com/nodegroup-image=ami-06abf956961f9530b,eks.amazonaws.com/sourceLaunchTemplateId=lt-0b52cbe4a1361f8c6,eks.amazonaws.com/sourceLaunchTemplateVersion=1 --register-with-taints  --pod-infra-container-image 602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1
   1272 ?        Sl     0:01 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96 -address /run/containerd/containerd.sock
   1273 ?        Sl     0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e -address /run/containerd/containerd.sock
   1637 ?        Sl     0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace default -id control -address /run/host-containerd/containerd.sock
   1674 ?        Sl     0:00 /x86_64-bottlerocket-linux-gnu/sys-root/usr/bin/containerd-shim-runc-v2 -namespace default -id admin -address /run/host-containerd/containerd.sock
  12370 ?        S+     0:00      |                           \_ grep containerd

# 당연 onpremise 에서 컨트롤플레인이 위치한 static pod 들은 보이지 않는다.
bash-5.1# ls -al /etc/kubernetes/manifests
lrwxrwxrwx. 1 root root 11 Apr 29 14:46 /etc/kubernetes/manifests -> static-pods

bash-5.1# ls -al /etc/kubernetes/manifests/
total 0
drwxr-xr-x. 2 root root  40 Apr 29 14:46 .
drwxr-xr-x. 6 root root 160 Apr 29 14:46 ..

```

### 노드 스토리지 정보 확인 

```sh 

bash-5.1# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0          7:0    0  296K  1 loop /x86_64-bottlerocket-linux-gnu/sys-root/usr/share/licenses
loop1          7:1    0 12.1M  1 loop /var/lib/kernel-devel/.overlay/lower
nvme0n1      259:0    0    2G  0 disk
|-nvme0n1p1  259:2    0    4M  0 part
|-nvme0n1p2  259:3    0    5M  0 part
|-nvme0n1p3  259:4    0   40M  0 part /boot
|-nvme0n1p4  259:5    0  920M  0 part
|-nvme0n1p5  259:6    0   10M  0 part
|-nvme0n1p6  259:7    0   25M  0 part
|-nvme0n1p7  259:8    0    5M  0 part
|-nvme0n1p8  259:9    0   40M  0 part
|-nvme0n1p9  259:10   0  920M  0 part
|-nvme0n1p10 259:11   0   10M  0 part
|-nvme0n1p11 259:12   0   25M  0 part
|-nvme0n1p12 259:13   0   41M  0 part /var/lib/bottlerocket
`-nvme0n1p13 259:14   0    1M  0 part
nvme1n1      259:1    0   30G  0 disk       # 우리가 eksctl 에서 명시한 eks 용 볼륨
`-nvme1n1p1  259:16   0   30G  0 part /var
                                      /opt
                                      /mnt
                                      /local
# ext4 로 구성이 되어있다. 
bash-5.1# df -hT
Filesystem      Type      Size  Used Avail Use% Mounted on
/dev/root       ext4      904M  640M  202M  77% /
devtmpfs        devtmpfs  1.9G     0  1.9G   0% /dev
tmpfs           tmpfs     1.9G     0  1.9G   0% /dev/shm
tmpfs           tmpfs     769M  744K  768M   1% /run
tmpfs           tmpfs     4.0M     0  4.0M   0% /sys/fs/cgroup
tmpfs           tmpfs     1.9G  480K  1.9G   1% /etc
tmpfs           tmpfs     1.9G  4.0K  1.9G   1% /etc/cni
tmpfs           tmpfs     1.9G     0  1.9G   0% /tmp
tmpfs           tmpfs     1.9G  8.0K  1.9G   1% /etc/containerd
tmpfs           tmpfs     1.9G   12K  1.9G   1% /etc/host-containers
tmpfs           tmpfs     1.9G     0  1.9G   0% /etc/kubernetes/pki/private
tmpfs           tmpfs     1.9G     0  1.9G   0% /root/.aws
/dev/nvme1n1p1  ext4       30G  1.3G   29G   5% /local
/dev/nvme0n1p12 ext4       35M  1.2M   31M   4% /var/lib/bottlerocket
overlay         overlay    30G  1.3G   29G   5% /opt/cni/bin
overlay         overlay    30G  1.3G   29G   5% /x86_64-bottlerocket-linux-gnu/sys-root/usr/lib/modules
/dev/loop1      squashfs   13M   13M     0 100% /var/lib/kernel-devel/.overlay/lower
/dev/loop0      squashfs  384K  384K     0 100% /x86_64-bottlerocket-linux-gnu/sys-root/usr/share/licenses
overlay         overlay    30G  1.3G   29G   5% /x86_64-bottlerocket-linux-gnu/sys-root/usr/src/kernels
/dev/nvme0n1p3  ext4       23M   14M  7.6M  64% /boot
tmpfs           tmpfs     3.3G   12K  3.3G   1% /var/lib/kubelet/pods/307f790a-dd46-4fb3-a8eb-1a834c75e222/volumes/kubernetes.io~projected/kube-api-access-8kql7
tmpfs           tmpfs     3.3G   12K  3.3G   1% /var/lib/kubelet/pods/25cac913-7c12-417b-b20e-f58347935b7b/volumes/kubernetes.io~projected/kube-api-access-44kxl
shm             tmpfs      64M     0   64M   0% /run/containerd/io.containerd.grpc.v1.cri/sandboxes/305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96/shm
shm             tmpfs      64M     0   64M   0% /run/containerd/io.containerd.grpc.v1.cri/sandboxes/93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e/shm
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/a89f25de856d8dd03fbe5a19655e15205e6cc01e6f5f2e8fe3a1febe4a70ddfb/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/host-containerd/io.containerd.runtime.v2.task/default/control/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/host-containerd/io.containerd.runtime.v2.task/default/admin/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/753fd61ca81f5a9279def3ef1266f16922c6923672c3b937d501f77c78676705/rootfs

# 특이한점으로는 etc, cni 등등 특정 파일들이 tmpfs 를 사용하고 있다
tmpfs           tmpfs     1.9G     0  1.9G   0% /dev/shm
tmpfs           tmpfs     769M  744K  768M   1% /run
tmpfs           tmpfs     4.0M     0  4.0M   0% /sys/fs/cgroup
tmpfs           tmpfs     1.9G  480K  1.9G   1% /etc
tmpfs           tmpfs     1.9G  4.0K  1.9G   1% /etc/cni
tmpfs           tmpfs     1.9G     0  1.9G   0% /tmp
tmpfs           tmpfs     1.9G  8.0K  1.9G   1% /etc/containerd
tmpfs           tmpfs     1.9G   12K  1.9G   1% /etc/host-containers
tmpfs           tmpfs     1.9G     0  1.9G   0% /etc/kubernetes/pki/private
tmpfs           tmpfs     1.9G     0  1.9G   0% /root/.aws

# overlay 로 30G 할당 되어있는거 보니 runtime 에서 추가 disk 등을 소비 하고 있는것으로 보인다. 
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/a89f25de856d8dd03fbe5a19655e15205e6cc01e6f5f2e8fe3a1febe4a70ddfb/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/host-containerd/io.containerd.runtime.v2.task/default/control/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/host-containerd/io.containerd.runtime.v2.task/default/admin/rootfs
overlay         overlay    30G  1.3G   29G   5% /run/containerd/io.containerd.runtime.v2.task/k8s.io/753fd61ca81f5a9279def3ef1266f16922c6923672c3b937d501f77c78676705/rootfs

```

### ctr 로 상태 확인 

```sh 

# ctr 로 기존에 조회 했던 내용들은 잘 조회 된다. 
bash-5.1# ctr ns list
NAME   LABELS
k8s.io

bash-5.1# ctr -n k8s.io container ls
CONTAINER                                                           IMAGE                                                                                          RUNTIME
305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96    602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1                     io.containerd.runc.v2
753fd61ca81f5a9279def3ef1266f16922c6923672c3b937d501f77c78676705    602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni:v1.11.4-eksbuild.1            io.containerd.runc.v2
93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e    602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1                     io.containerd.runc.v2
a89f25de856d8dd03fbe5a19655e15205e6cc01e6f5f2e8fe3a1febe4a70ddfb    602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/kube-proxy:v1.24.7-minimal-eksbuild.2    io.containerd.runc.v2
d908e888c1f1bf95709569b145fcb98ade90655dae73c26af0fe89813e672680    602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni-init:v1.11.4-eksbuild.1       io.containerd.runc.v2

bash-5.1# ctr -n k8s.io i ls
REF                                                                                                                                           TYPE                                                      DIGEST                                                                  SIZE      PLATFORMS                  LABELS
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni-init:v1.11.4-eksbuild.1                                                      application/vnd.docker.distribution.manifest.list.v2+json sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd 96.6 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni-init@sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd application/vnd.docker.distribution.manifest.list.v2+json sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd 96.6 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni:v1.11.4-eksbuild.1                                                           application/vnd.docker.distribution.manifest.list.v2+json sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4 97.0 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/amazon-k8s-cni@sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4      application/vnd.docker.distribution.manifest.list.v2+json sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4 97.0 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/kube-proxy:v1.24.7-minimal-eksbuild.2                                                   application/vnd.docker.distribution.manifest.list.v2+json sha256:a4f32ea2ddf8d47424b6f4e9902ceabe6f449afbeaca843c7803dbf541f0837c 24.4 MiB  linux/amd64,linux/arm64    io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/kube-proxy@sha256:a4f32ea2ddf8d47424b6f4e9902ceabe6f449afbeaca843c7803dbf541f0837c      application/vnd.docker.distribution.manifest.list.v2+json sha256:a4f32ea2ddf8d47424b6f4e9902ceabe6f449afbeaca843c7803dbf541f0837c 24.4 MiB  linux/amd64,linux/arm64    io.cri-containerd.image=managed
602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.1-eksbuild.1                                                                    application/vnd.docker.distribution.manifest.list.v2+json sha256:1cb4ab85a3480446f9243178395e6bee7350f0d71296daeb6a9fdd221e23aea6 292.4 KiB linux/amd64,linux/arm64    io.cri-containerd.image=managed
ecr.aws/arn:aws:ecr:ap-northeast-3:602401143452:repository/eks/pause:3.1-eksbuild.1                                                           application/vnd.docker.distribution.manifest.list.v2+json sha256:1cb4ab85a3480446f9243178395e6bee7350f0d71296daeb6a9fdd221e23aea6 292.4 KiB linux/amd64,linux/arm64    io.cri-containerd.image=managed
sha256:04beb3b811d345722d689a70a30bafa27e0edd412613bee76c3648b024b25744                                                                       application/vnd.docker.distribution.manifest.list.v2+json sha256:a4f32ea2ddf8d47424b6f4e9902ceabe6f449afbeaca843c7803dbf541f0837c 24.4 MiB  linux/amd64,linux/arm64    io.cri-containerd.image=managed
sha256:106a8e54d5eb3f70fcd1ed46255bdf232b3f169e89e68e13e4e67b25f59c1315                                                                       application/vnd.docker.distribution.manifest.list.v2+json sha256:1cb4ab85a3480446f9243178395e6bee7350f0d71296daeb6a9fdd221e23aea6 292.4 KiB linux/amd64,linux/arm64    io.cri-containerd.image=managed
sha256:41626cc0545b23c4459cd892685cf66ec048ae5b7113a6efc29db46c114b2159                                                                       application/vnd.docker.distribution.manifest.list.v2+json sha256:e3cb52e6e9b054c7638e0015637ac07a44bd5a9bf5dd4ac98a40c57ce21544fd 96.6 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed
sha256:5bad0186aac16adcf75b396775c9605ce4bad340fe641fce18b0be046558fb73                                                                       application/vnd.docker.distribution.manifest.list.v2+json sha256:2fd65a8764b71c2f3ae6ef89e5cb0d85f57f52514815b2a9a40ccad582743fa4 97.0 MiB  linux/amd64,linux/arm64/v8 io.cri-containerd.image=managed

bash-5.1# ctr -n k8s.io t ls
TASK                                                                PID     STATUS
305032b9996e9171c48c069c953e9e9029051be6b46a99a07adc54db1df0cf96    1337    RUNNING
93b5333909557bbc94b74fe395f4e6e8a80e90a7b4e689a7b40ed7bc9dfd158e    1332    RUNNING
a89f25de856d8dd03fbe5a19655e15205e6cc01e6f5f2e8fe3a1febe4a70ddfb    1397    RUNNING
753fd61ca81f5a9279def3ef1266f16922c6923672c3b937d501f77c78676705    1903    RUNNING

bash-5.1# ps -c 1337
    PID CLS PRI TTY      STAT   TIME COMMAND
   1337 TS   19 ?        Ss     0:00 /pause
bash-5.1# ps -c 1332
    PID CLS PRI TTY      STAT   TIME COMMAND
   1332 TS   19 ?        Ss     0:00 /pause
bash-5.1# ps -c 1397
    PID CLS PRI TTY      STAT   TIME COMMAND
   1397 TS   19 ?        Ssl    0:00 kube-proxy --v=2 --config=/var/lib/kube-proxy-config/config --hostname-override=ip-192-168-1-120.ap-northeast-3.compute.internal
bash-5.1# ps -c 1903
    PID CLS PRI TTY      STAT   TIME COMMAND
   1903 TS   19 ?        Ss     0:00 bash /app/entrypoint.sh

```

* 시간을 두고 좀 더 보고 싶기는 한데 아쉽지만 이정도로 마무리 하고자 한다. 


### 생성한 eks cluster 삭제 

* eksctl delete cluster --name eks-bottlerocket

## cloudformation 삭제 

* aws cloudformation delete-stack --stack-name myeks
