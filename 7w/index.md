---
title: "EKS Automation "
date: 2023-06-02T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 7w
    identifier: 7w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 

* Cloudnet 에서 진행하는 AEWS 스터디 7주차 Automation 이다.
* AWS Controller for Kubernetes (ACK) 를 이용하여서 AWS Resources 를 관리 하는 방법 등을 학습 할 예정이다. 
* 아마 argoCD, Flux 등은 스터디에서는 다루지만 나는 별도 기술하지 않을 것이다.

## 실습 환경 

```sh 
# YAML 파일 다운로드
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick6.yaml

# CloudFormation 스택 배포
예시) aws cloudformation deploy --template-file eks-oneclick5.yaml --stack-name myeks --parameter-overrides KeyName=kp-gasida SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32  MyIamUserAccessKeyID=AKIA5... MyIamUserSecretAccessKey='CVNa2...' ClusterBaseName=myeks --region ap-northeast-2

# CloudFormation 스택 배포 완료 후 작업용 EC2 IP 출력
aws cloudformation describe-stacks --stack-name myeks --query 'Stacks[*].Outputs[0].OutputValue' --output text

# 작업용 EC2 SSH 접속
ssh -i ~/.ssh/kp-gasida.pem ec2-user@$(aws cloudformation describe-stacks --stack-name myeks --query 'Stacks[*].Outputs[0].OutputValue' --output text)


# 기본 설정 
# default 네임스페이스 적용
kubectl ns default

# (옵션) context 이름 변경
NICK=<각자 자신의 닉네임>
NICK=gasida
kubectl ctx
kubectl config rename-context admin@myeks.ap-northeast-2.eksctl.io $NICK

# ExternalDNS
MyDomain=<자신의 도메인>
echo "export MyDomain=<자신의 도메인>" >> /etc/profile
MyDomain=gasida.link
echo "export MyDomain=gasida.link" >> /etc/profile
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -

# AWS LB Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

# 노드 IP 확인 및 PrivateIP 변수 지정
N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2b -o jsonpath={.items[0].status.addresses[0].address})
N3=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2c -o jsonpath={.items[0].status.addresses[0].address})
echo "export N1=$N1" >> /etc/profile
echo "export N2=$N2" >> /etc/profile
echo "export N3=$N3" >> /etc/profile
echo $N1, $N2, $N3

# 노드 보안그룹 ID 확인
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values='*ng1*' --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.0/24

# 워커 노드 SSH 접속
for node in $N1 $N2 $N3; do ssh ec2-user@$node hostname; done
```

## AWS Controller for Kubernetes 

> AWS 리소스를 K8S CRD 를 이용하여 직접 정의 하고 사용할수 있는 K8s Controller 라고 보면 될듯하다. 
> AWS 리소스의 예시는 S3, VPC 등을 이야기 한다. 

![ACK-Diagram](/7w/images/ack-diagram.png)

* ACK 는 AWS 리소스 별로 Controller 가 구성이 되어있으며 각 리소스 별로 어떤 컨트롤러는 GA, 어떤 컨트롤러는 Preview 상태로 되어있다. 
* [서비스 상태](https://aws-controllers-k8s.github.io/community/docs/community/services/) 는 링크를 확인해서 사용이 가능한가 판단 하면 될듯하다.
* 권한에 대하여는 아래와 같은 그림으로 권한을 요청하고 있다.

![ACK-perm-Diagram](/7w/images/ack-permission-diagram.png)

### ACK S3 Controller 설치 

```sh
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export SERVICE=s3
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export RELEASE_VERSION=$(curl -sL https://api.github.com/repos/aws-controllers-k8s/$SERVICE-controller/releases/latest | grep '"tag_name":' | cut -d'"' -f4 | cut -c 2-)
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# helm pull oci://public.ecr.aws/aws-controllers-k8s/$SERVICE-chart --version=$RELEASE_VERSION
Pulled: public.ecr.aws/aws-controllers-k8s/s3-chart:1.0.4
Digest: sha256:9cd8574c78c7f226a2520a423a447afd02366a3ec87b5d1ba910992da3e264b8
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# tar xzvf $SERVICE-chart-$RELEASE_VERSION.tgz
s3-chart/Chart.yaml
s3-chart/values.yaml
s3-chart/values.schema.json
s3-chart/templates/NOTES.txt
s3-chart/templates/_helpers.tpl
s3-chart/templates/cluster-role-binding.yaml
s3-chart/templates/cluster-role-controller.yaml
s3-chart/templates/deployment.yaml
s3-chart/templates/metrics-service.yaml
s3-chart/templates/role-reader.yaml
s3-chart/templates/role-writer.yaml
s3-chart/templates/service-account.yaml
s3-chart/crds/s3.services.k8s.aws_buckets.yaml
s3-chart/crds/services.k8s.aws_adoptedresources.yaml
s3-chart/crds/services.k8s.aws_fieldexports.yaml
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# tree ~/$SERVICE-chart
/root/s3-chart
├── Chart.yaml
├── crds
│   ├── s3.services.k8s.aws_buckets.yaml
│   ├── services.k8s.aws_adoptedresources.yaml
│   └── services.k8s.aws_fieldexports.yaml
├── templates
│   ├── cluster-role-binding.yaml
│   ├── cluster-role-controller.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── metrics-service.yaml
│   ├── NOTES.txt
│   ├── role-reader.yaml
│   ├── role-writer.yaml
│   └── service-account.yaml
├── values.schema.json
└── values.yaml

2 directories, 15 files
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export ACK_SYSTEM_NAMESPACE=ack-system
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export AWS_REGION=ap-northeast-2
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# helm install --create-namespace -n $ACK_SYSTEM_NAMESPACE ack-$SERVICE-controller --set aws.region="$AWS_REGION" ~/$SERVICE-chart
NAME: ack-s3-controller
LAST DEPLOYED: Sat Jun 10 17:57:46 2023
NAMESPACE: ack-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
s3-chart has been installed.
This chart deploys "public.ecr.aws/aws-controllers-k8s/s3-controller:1.0.4".

Check its status by running:
  kubectl --namespace ack-system get pods -l "app.kubernetes.io/instance=ack-s3-controller"

You are now able to create Amazon Simple Storage Service (S3) resources!

The controller is running in "cluster" mode.
The controller is configured to manage AWS resources in region: "ap-northeast-2"

Visit https://aws-controllers-k8s.github.io/community/reference/ for an API
reference of all the resources that can be created using this controller.

For more information on the AWS Controllers for Kubernetes (ACK) project, visit:
https://aws-controllers-k8s.github.io/community/

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# helm list --namespace $ACK_SYSTEM_NAMESPACE
NAME             	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
ack-s3-controller	ack-system	1       	2023-06-10 17:57:46.541298067 +0900 KST	deployed	s3-chart-1.0.4	1.0.4

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl -n ack-system get pods
NAME                                          READY   STATUS    RESTARTS   AGE
ack-s3-controller-s3-chart-7c55c6657d-bb6q9   1/1     Running   0          40s

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl get crd | grep $SERVICE
buckets.s3.services.k8s.aws                  2023-06-10T08:57:46Z

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl get all -n ack-system
NAME                                              READY   STATUS    RESTARTS   AGE
pod/ack-s3-controller-s3-chart-7c55c6657d-bb6q9   1/1     Running   0          65s

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ack-s3-controller-s3-chart   1/1     1            1           65s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/ack-s3-controller-s3-chart-7c55c6657d   1         1         1       65s

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl get-all -n ack-system
NAME                                                   NAMESPACE   AGE
configmap/kube-root-ca.crt                             ack-system  67s
pod/ack-s3-controller-s3-chart-7c55c6657d-bb6q9        ack-system  67s
secret/sh.helm.release.v1.ack-s3-controller.v1         ack-system  67s
serviceaccount/ack-s3-controller                       ack-system  67s
serviceaccount/default                                 ack-system  67s
deployment.apps/ack-s3-controller-s3-chart             ack-system  67s
replicaset.apps/ack-s3-controller-s3-chart-7c55c6657d  ack-system  67s
role.rbac.authorization.k8s.io/ack-s3-reader           ack-system  67s
role.rbac.authorization.k8s.io/ack-s3-writer           ack-system  67s

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl describe sa -n ack-system ack-s3-controller
Name:                ack-s3-controller
Namespace:           ack-system
Labels:              app.kubernetes.io/instance=ack-s3-controller
                     app.kubernetes.io/managed-by=Helm
                     app.kubernetes.io/name=s3-chart
                     app.kubernetes.io/version=1.0.4
                     helm.sh/chart=s3-chart-1.0.4
                     k8s-app=s3-chart
Annotations:         meta.helm.sh/release-name: ack-s3-controller
                     meta.helm.sh/release-namespace: ack-system
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>

# 배포할 S3 의 권한을 부여하기 위한 IRSA 를 구성 해준다. 
eksctl create iamserviceaccount \
  --name ack-$SERVICE-controller \
  --namespace ack-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3FullAccess`].Arn' --output text) \
  --override-existing-serviceaccounts --approve

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# eksctl get iamserviceaccount --cluster $CLUSTER_NAME
NAMESPACE	NAME				ROLE ARN
ack-system	ack-s3-controller		arn:aws:iam::123123123:role/eksctl-myeks-addon-iamserviceaccount-ack-sys-Role1-YNTAFQU2F6CC
kube-system	aws-load-balancer-controller	arn:aws:iam::123123123:role/eksctl-myeks-addon-iamserviceaccount-kube-sy-Role1-C5T18T1WEM2W

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl -n ack-system rollout restart deploy ack-$SERVICE-controller-$SERVICE-chart
deployment.apps/ack-s3-controller-s3-chart restarted

```


### ACK 를 이용하여 S3 버킷 생성 업데이트 예시

```sh 
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# export BUCKET_NAME=my-ack-s3-bucket-$AWS_ACCOUNT_ID

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# read -r -d '' BUCKET_MANIFEST <<EOF
> apiVersion: s3.services.k8s.aws/v1alpha1
> kind: Bucket
> metadata:
>   name: $BUCKET_NAME
> spec:
>   name: $BUCKET_NAME
> EOF

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# echo "${BUCKET_MANIFEST}" > bucket.yaml

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# cat bucket.yaml | yh
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: my-ack-s3-bucket-123123123
spec:
  name: my-ack-s3-bucket-123123123


(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# aws s3 ls
2023-03-05 12:22:32 se4ofnight-k8s-s3

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl create -f bucket.yaml
bucket.s3.services.k8s.aws/my-ack-s3-bucket-123123123 created

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# aws s3 ls
2023-06-10 18:05:03 my-ack-s3-bucket-123123123

# 버킷 확인 
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl get buckets
NAME                            AGE
my-ack-s3-bucket-123123123   56s

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl describe bucket/$BUCKET_NAME | head -6
Name:         my-ack-s3-bucket-123123123
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  s3.services.k8s.aws/v1alpha1
Kind:         Bucket

# 버킷 CRD 변경시 aws s3 적용 되는지 확인 
read -r -d '' BUCKET_MANIFEST <<EOF
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: $BUCKET_NAME
spec:
  name: $BUCKET_NAME
  tagging:
    tagSet:
    - key: myTagKey
      value: myTagValue
EOF

echo "${BUCKET_MANIFEST}" > bucket.yaml

(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl apply -f bucket.yaml

# 변경한 정보 확인 
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl describe bucket/$BUCKET_NAME | grep Spec: -A5
Spec:
  Name:  my-ack-s3-bucket-123123123
Status:
  Ack Resource Metadata:
    Owner Account ID:  123123123
    Region:            ap-northeast-2

# 버킷 삭제 
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# kubectl delete -f bucket.yaml
bucket.s3.services.k8s.aws "my-ack-s3-bucket-123123123" deleted
(se4ofnight@myeks:N/A) [root@myeks-bastion ~]# aws s3 ls
# 없음
```