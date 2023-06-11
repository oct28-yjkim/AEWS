---
title: "EKS Security "
date: 2023-06-02T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 6w
    identifier: 6w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 

* Cloudnet 에서 진행하는 AEWS 스터디 6주차 Security 이다. 
* K8s, EKS 의 인증 인가 등에 다루게 되며 
* IRSA 등을 실습 하게 될듯하다. 


## 실습 환경 

```sh 
# YAML 파일 다운로드
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick5.yaml

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


## EKS 인증 인가 

* 사용자 / 애플리케이션 -> K8S 사용시 : 인증은 AWS ISM, 인가는 K8S RBAC



```sh 
kubectl krew install access-matrix rbac-tool rbac-view rolesum

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl access-matrix # Review access to cluster-scoped resources
NAME                                                          LIST  CREATE  UPDATE  DELETE
...
                                                              ✖     ✖       ✖       ✖
apiservices.apiregistration.k8s.io                            ✔     ✔       ✔       ✔
bindings                                                            ✔
certificatesigningrequests.certificates.k8s.io                ✔     ✔       ✔       ✔
clusterrolebindings.rbac.authorization.k8s.io                 ✔     ✔       ✔       ✔
clusterroles.rbac.authorization.k8s.io                        ✔     ✔       ✔       ✔
componentstatuses                                             ✔
configmaps                                                    ✔     ✔       ✔       ✔
controllerrevisions.apps                                      ✔     ✔       ✔       ✔
cronjobs.batch                                                ✔     ✔       ✔       ✔
csidrivers.storage.k8s.io                                     ✔     ✔       ✔       ✔
csinodes.storage.k8s.io                                       ✔     ✔       ✔       ✔
csistoragecapacities.storage.k8s.io                           ✔     ✔       ✔       ✔
customresourcedefinitions.apiextensions.k8s.io                ✔     ✔       ✔       ✔
daemonsets.apps                                               ✔     ✔       ✔       ✔
deployments.apps                                              ✔     ✔       ✔       ✔
endpoints                                                     ✔     ✔       ✔       ✔
endpointslices.discovery.k8s.io                               ✔     ✔       ✔       ✔
eniconfigs.crd.k8s.amazonaws.com                              ✔     ✔       ✔       ✔
events                                                        ✔     ✔       ✔       ✔
events.events.k8s.io                                          ✔     ✔       ✔       ✔
flowschemas.flowcontrol.apiserver.k8s.io                      ✔     ✔       ✔       ✔
horizontalpodautoscalers.autoscaling                          ✔     ✔       ✔       ✔
ingressclasses.networking.k8s.io                              ✔     ✔       ✔       ✔
ingresses.networking.k8s.io                                   ✔     ✔       ✔       ✔
jobs.batch                                                    ✔     ✔       ✔       ✔
leases.coordination.k8s.io                                    ✔     ✔       ✔       ✔
limitranges                                                   ✔     ✔       ✔       ✔
localsubjectaccessreviews.authorization.k8s.io                      ✔
mutatingwebhookconfigurations.admissionregistration.k8s.io    ✔     ✔       ✔       ✔
namespaces                                                    ✔     ✔       ✔       ✔
networkpolicies.networking.k8s.io                             ✔     ✔       ✔       ✔
nodes                                                         ✔     ✔       ✔       ✔
persistentvolumeclaims                                        ✔     ✔       ✔       ✔
persistentvolumes                                             ✔     ✔       ✔       ✔
poddisruptionbudgets.policy                                   ✔     ✔       ✔       ✔
pods                                                          ✔     ✔       ✔       ✔
podsecuritypolicies.policy                                    ✔     ✔       ✔       ✔
podtemplates                                                  ✔     ✔       ✔       ✔
priorityclasses.scheduling.k8s.io                             ✔     ✔       ✔       ✔
prioritylevelconfigurations.flowcontrol.apiserver.k8s.io      ✔     ✔       ✔       ✔
replicasets.apps                                              ✔     ✔       ✔       ✔
replicationcontrollers                                        ✔     ✔       ✔       ✔
resourcequotas                                                ✔     ✔       ✔       ✔
rolebindings.rbac.authorization.k8s.io                        ✔     ✔       ✔       ✔
roles.rbac.authorization.k8s.io                               ✔     ✔       ✔       ✔
runtimeclasses.node.k8s.io                                    ✔     ✔       ✔       ✔
secrets                                                       ✔     ✔       ✔       ✔
securitygrouppolicies.vpcresources.k8s.aws                    ✔     ✔       ✔       ✔
selfsubjectaccessreviews.authorization.k8s.io                       ✔
selfsubjectrulesreviews.authorization.k8s.io                        ✔
serviceaccounts                                               ✔     ✔       ✔       ✔
services                                                      ✔     ✔       ✔       ✔
statefulsets.apps                                             ✔     ✔       ✔       ✔
storageclasses.storage.k8s.io                                 ✔     ✔       ✔       ✔
subjectaccessreviews.authorization.k8s.io                           ✔
tokenreviews.authentication.k8s.io                                  ✔
validatingwebhookconfigurations.admissionregistration.k8s.io  ✔     ✔       ✔       ✔
volumeattachments.storage.k8s.io                              ✔     ✔       ✔       ✔
No namespace given, this implies cluster scope (try -n if this is not intended)

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl access-matrix --namespace default # Review access to namespaced resources in 'default'
NAME                                            LIST  CREATE  UPDATE  DELETE
...
                                                ✖     ✖       ✖       ✖
bindings                                              ✔
configmaps                                      ✔     ✔       ✔       ✔
controllerrevisions.apps                        ✔     ✔       ✔       ✔
cronjobs.batch                                  ✔     ✔       ✔       ✔
csistoragecapacities.storage.k8s.io             ✔     ✔       ✔       ✔
daemonsets.apps                                 ✔     ✔       ✔       ✔
deployments.apps                                ✔     ✔       ✔       ✔
endpoints                                       ✔     ✔       ✔       ✔
endpointslices.discovery.k8s.io                 ✔     ✔       ✔       ✔
events                                          ✔     ✔       ✔       ✔
events.events.k8s.io                            ✔     ✔       ✔       ✔
horizontalpodautoscalers.autoscaling            ✔     ✔       ✔       ✔
ingresses.networking.k8s.io                     ✔     ✔       ✔       ✔
jobs.batch                                      ✔     ✔       ✔       ✔
leases.coordination.k8s.io                      ✔     ✔       ✔       ✔
limitranges                                     ✔     ✔       ✔       ✔
localsubjectaccessreviews.authorization.k8s.io        ✔
networkpolicies.networking.k8s.io               ✔     ✔       ✔       ✔
persistentvolumeclaims                          ✔     ✔       ✔       ✔
poddisruptionbudgets.policy                     ✔     ✔       ✔       ✔
pods                                            ✔     ✔       ✔       ✔
podtemplates                                    ✔     ✔       ✔       ✔
replicasets.apps                                ✔     ✔       ✔       ✔
replicationcontrollers                          ✔     ✔       ✔       ✔
resourcequotas                                  ✔     ✔       ✔       ✔
rolebindings.rbac.authorization.k8s.io          ✔     ✔       ✔       ✔
roles.rbac.authorization.k8s.io                 ✔     ✔       ✔       ✔
secrets                                         ✔     ✔       ✔       ✔
securitygrouppolicies.vpcresources.k8s.aws      ✔     ✔       ✔       ✔
serviceaccounts                                 ✔     ✔       ✔       ✔
services                                        ✔     ✔       ✔       ✔
statefulsets.apps                               ✔     ✔       ✔       ✔

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl rbac-tool lookup
W0603 18:42:05.291773    4222 warnings.go:67] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
  SUBJECT                            | SUBJECT TYPE   | SCOPE       | NAMESPACE   | ROLE
+------------------------------------+----------------+-------------+-------------+------------------------------------------------------+
  attachdetach-controller            | ServiceAccount | ClusterRole |             | system:controller:attachdetach-controller
  aws-node                           | ServiceAccount | ClusterRole |             | aws-node
  bootstrap-signer                   | ServiceAccount | Role        | kube-public | system:controller:bootstrap-signer
  bootstrap-signer                   | ServiceAccount | Role        | kube-system | system:controller:bootstrap-signer
  certificate-controller             | ServiceAccount | ClusterRole |             | system:controller:certificate-controller
  cloud-provider                     | ServiceAccount | Role        | kube-system | system:controller:cloud-provider
...

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl rbac-tool lookup system:masters
W0603 18:42:07.710551    4275 warnings.go:67] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
  SUBJECT        | SUBJECT TYPE | SCOPE       | NAMESPACE | ROLE
+----------------+--------------+-------------+-----------+---------------+
  system:masters | Group        | ClusterRole |           | cluster-admin

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl rbac-tool policy-rules | head -n 10
W0603 18:44:48.617584    4800 warnings.go:67] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
  TYPE           | SUBJECT                            | VERBS            | NAMESPACE   | API GROUP                 | KIND                                      | NAMES                                                                                                                                                                                            | NONRESOURCEURI                                                                           | ORIGINATED FROM
+----------------+------------------------------------+------------------+-------------+---------------------------+-------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------+--------------------------------------------------------------------+
  Group          | eks:kube-proxy-windows             | create           | *           | core                      | events                                    |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | create           | *           | events.k8s.io             | events                                    |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | get              | *           | core                      | nodes                                     |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | list             | *           | core                      | endpoints                                 |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | list             | *           | core                      | nodes                                     |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | list             | *           | core                      | services                                  |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | list             | *           | discovery.k8s.io          | endpointslices                            |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier
  Group          | eks:kube-proxy-windows             | patch            | *           | core                      | events                                    |                                                                                                                                                                                                  |                                                                                          | ClusterRoles>>system:node-proxier

  (yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl rbac-tool whoami
{Username: "kubernetes-admin",
 UID:      "aws-iam-authenticator:123123123:AIDAXFX...",
 Groups:   ["system:masters",
            "system:authenticated"],
 Extra:    {accessKeyId:  ["AKIAXFXFAA....."],
            arn:          ["arn:aws:iam::123123123:user/yjkim"],
            canonicalArn: ["arn:aws:iam::123123123:user/yjkim"],
            principalId:  ["AIDAXFXFAAO..."],
            sessionName:  [""]}}

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl rolesum aws-node -n kube-system
ServiceAccount: kube-system/aws-node
Secrets:

Policies:

• [CRB] */aws-node ⟶  [CR] */aws-node
  Resource                          Name  Exclude  Verbs  G L W C U P D DC
  *.extensions                      [*]     [-]     [-]   ✖ ✔ ✔ ✖ ✖ ✖ ✖ ✖
  eniconfigs.crd.k8s.amazonaws.com  [*]     [-]     [-]   ✔ ✔ ✔ ✖ ✖ ✖ ✖ ✖
  events.[,events.k8s.io]           [*]     [-]     [-]   ✖ ✔ ✖ ✔ ✖ ✔ ✖ ✖
  namespaces                        [*]     [-]     [-]   ✔ ✔ ✔ ✖ ✖ ✖ ✖ ✖
  nodes                             [*]     [-]     [-]   ✔ ✔ ✔ ✖ ✔ ✖ ✖ ✖
  pods                              [*]     [-]     [-]   ✔ ✔ ✔ ✖ ✖ ✖ ✖ ✖

```

* AWS EKS - 인증 인가 
![AWS EKS - 인증 인가 ](/posts/study/cloudnet-aews/6w/images/eks-auth.png)

* IAM Authenticator
![IAM Authenticator](/posts/study/cloudnet-aews/6w/images/iam-authenticator.png)

* kubectl 명령시 -> aws eks get-token -> EKS Services endpoint 에서 token 요청 
* kubectl 의 client-go 는 pre-signed url 을 bare token 으로 EKS API 클러스터 endpoint 로 요청을 보냄 
  * token을 jwt 사이트에서 복붙으로 디코드 된 정보 확인 가능 HS384 -> HS256
  * PAYLOAD 값을 URL decode 에서 확인 가능 
* EKS 에서는 Token Review 를 Webhook token authenticator 에 요청 
  * IAM 에 호출인증 완료후 User/Role 에 대한 ARN 반환 
```sh
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws sts get-caller-identity --query Arn
"arn:aws:iam::123123123:user/yjkim"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# cat ~/.kube/config | yh
apiVersion: v1
clusters:
- cluster:
...
users:
- name: yjkim@yjkim-eks.ap-northeast-3.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --output
      - json
      - --cluster-name
      - yjkim-eks
      - --region
      - ap-northeast-3
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      provideClusterInfo: false

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws eks get-token --cluster-name $CLUSTER_NAME | jq
{
  "kind": "ExecCredential",
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "spec": {},
  "status": {
    "expirationTimestamp": "2023-06-03T10:08:34Z",
    "token": "k8s-aws-v1
...

# token  decode example 
https://sts.ap-northeast-2.amazonaws.com/?
  Action=GetCallerIdentity&
  Version=2011-06-15&
  X-Amz-Algorithm=AWS4-HMAC-SHA256&
  X-Amz-Credential=AKIA5ILF.../20230525/ap-northeast-2/sts/aws4_request&
  X-Amz-Date=20230525T120720Z&
  X-Amz-Expires=60&
  X-Amz-SignedHeaders=host;x-k8s-aws-id&
  X-Amz-Signature=6e09b846da702767f38c78831986cb558.....

# token reviews api 리소스 
kubectl api-resources | grep authentication
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview

# List the fields for supported resources.
kubectl explain tokenreviews
...
DESCRIPTION:
     TokenReview attempts to authenticate a token to a known user. Note:
     TokenReview requests may be cached by the webhook token authenticator
     plugin in the kube-apiserver.

```


* 인증이 처리되면 k8s 에서 RBAC 인가 처리 

```sh 
# Webhook api 리소스 확인 
kubectl api-resources | grep Webhook
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration

# validatingwebhookconfigurations 리소스 확인
kubectl get validatingwebhookconfigurations
NAME                                        WEBHOOKS   AGE
eks-aws-auth-configmap-validation-webhook   1          50m
vpc-resource-validating-webhook             2          50m
aws-load-balancer-webhook                   3          8m27s

kubectl get validatingwebhookconfigurations eks-aws-auth-configmap-validation-webhook -o yaml | kubectl neat | yh

# aws-auth 컨피그맵 확인
kubectl get cm -n kube-system aws-auth -o yaml | kubectl neat | yh
apiVersion: v1
kind: ConfigMap
metadata: 
  name: aws-auth
  namespace: kube-system
data: 
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::91128.....:role/eksctl-myeks-nodegroup-ng1-NodeInstanceRole-1OS1WSTV0YB9X
      username: system:node:{{EC2PrivateDNSName}}
#---<아래 생략(추정), ARN은 EKS를 설치한 IAM User , 여기 있었을경우 만약 실수로 삭제 시 복구가 가능했을까?---
  mapUsers: |
    - groups:
      - system:masters
      userarn: arn:aws:iam::111122223333:user/admin
      username: kubernetes-admin

# EKS 설치한 IAM User 정보 >> system:authenticated는 어떤 방식으로 추가가 되었는지 궁금???
kubectl rbac-tool whoami
{Username: "kubernetes-admin",
 UID:      "aws-iam-authenticator:9112834...:AIDA5ILF2FJIR2.....",
 Groups:   ["system:masters",
            "system:authenticated"],
...

# system:masters , system:authenticated 그룹의 정보 확인
kubectl rbac-tool lookup system:masters
kubectl rbac-tool lookup system:authenticated
kubectl rolesum -k Group system:masters
kubectl rolesum -k Group system:authenticated

# system:masters 그룹이 사용 가능한 클러스터 롤 확인 : cluster-admin
kubectl describe clusterrolebindings.rbac.authorization.k8s.io cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters

# cluster-admin 의 PolicyRule 확인 : 모든 리소스  사용 가능!
kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]

# system:authenticated 그룹이 사용 가능한 클러스터 롤 확인
kubectl describe ClusterRole system:discovery
kubectl describe ClusterRole system:public-info-viewer
kubectl describe ClusterRole system:basic-user
kubectl describe ClusterRole eks:podsecuritypolicy:privileged
```

### 시나리오 : EKS 에 자격증명 적용해보기 

* AWS 테스트 유저 생성
* 자격증명 설정 
* 유저를 EKS 에 권한 설정 
* kubeconfig 적용 
* rbac 적용 확인 
* aws-auth configmap 확인 

```sh 
# 테스트 계정 생성
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws iam create-user --user-name testuser
{
    "User": {
        "Path": "/",
        "UserName": "testuser",
        "UserId": "AIDAXFX...",
        "Arn": "arn:aws:iam::123123123:user/testuser",
        "CreateDate": "2023-06-04T02:37:15+00:00"
    }
}
# api 오쳥 활성화 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws iam create-access-key --user-name testuser
{
    "AccessKey": {
        "UserName": "testuser",
        "AccessKeyId": "AKIAXFXFAA...",
        "Status": "Active",
        "SecretAccessKey": "GbbC2781uZrahcuaunK2K0MLvSSn2kTNzw5EcZJS",
        "CreateDate": "2023-06-04T02:37:20+00:00"
    }
}

# 권한 부여 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --user-name testuser

# arn 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws sts get-caller-identity --query Arn
"arn:aws:iam::123123123:user/yjkim"

# ec2 public ip 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
---------------------------------------------------------------------------
|                            DescribeInstances                            |
+--------------------------+----------------+-----------------+-----------+
|       InstanceName       | PrivateIPAdd   |   PublicIPAdd   |  Status   |
+--------------------------+----------------+-----------------+-----------+
|  yjkim-eks-ng1-Node      |  192.168.3.249 |  15.152.110.137 |  running  |
|  yjkim-eks-ng1-Node      |  192.168.2.86  |  15.152.93.129  |  running  |
|  yjkim-eks-ng1-Node      |  192.168.1.70  |  15.168.37.154  |  running  |
|  yjkim-eks-bastion-EC2-2 |  192.168.1.200 |  13.208.211.204 |  running  |
|  yjkim-eks-bastion-EC2   |  192.168.1.100 |  13.208.211.175 |  running  |
+--------------------------+----------------+-----------------+-----------+

# kubeconfig 에 자격 증명 설정 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws sts get-caller-identity --query Arn
"arn:aws:iam::123123123:user/yjkim"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws configure
Default output format [None]:

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws sts get-caller-identity --query Arn
"arn:aws:iam::123123123:user/yjkim"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl get node -v6
I0604 11:40:47.487187    8226 loader.go:374] Config loaded from file:  /root/.kube/config
I0604 11:40:48.251437    8226 round_trippers.go:553] GET https://45833AC6216CC4B071830959E68507F8.gr7.ap-northeast-3.eks.amazonaws.com/api/v1/nodes?limit=500 200 OK in 757 milliseconds
NAME                                               STATUS   ROLES    AGE   VERSION
ip-192-168-1-70.ap-northeast-3.compute.internal    Ready    <none>   17h   v1.24.13-eks-0a21954
ip-192-168-2-86.ap-northeast-3.compute.internal    Ready    <none>   17h   v1.24.13-eks-0a21954
ip-192-168-3-249.ap-northeast-3.compute.internal   Ready    <none>   17h   v1.24.13-eks-0a21954

# eks 에 자격 증명 구성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# eksctl create iamidentitymapping --cluster $CLUSTER_NAME --username testuser --group system:masters --arn arn:aws:iam::$ACCOUNT_ID:user/testuser
2023-06-04 11:42:03 [ℹ]  checking arn arn:aws:iam::123123123:user/testuser against entries in the auth ConfigMap
2023-06-04 11:42:03 [ℹ]  adding identity "arn:aws:iam::123123123:user/testuser" to auth ConfigMap

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# kubectl get cm -n kube-system aws-auth -o yaml | kubectl neat | yh
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-ng1-NodeInstanceRole-1J7PRS1TBQQML
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - groups:
      - system:masters
      userarn: arn:aws:iam::123123123:user/testuser
      username: testuser
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# eksctl get iamidentitymapping --cluster $CLUSTER_NAME
ARN												USERNAME				GROUPS					ACCOUNT
arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-ng1-NodeInstanceRole-1J7PRS1TBQQML	system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes
arn:aws:iam::123123123:user/testuser								testuser				system:masters

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion ~]# aws eks update-kubeconfig --name $CLUSTER_NAME --user-alias testuser
Added new context testuser to /root/.kube/config

(testuser:N/A) [root@yjkim-eks-bastion ~]# cat ~/.kube/config | yh
apiVersion: v1
clusters:
- cluster:
...
  name: yjkim-eks.ap-northeast-3.eksctl.io
- cluster:
...
contexts:
- context:
...
- context:
    cluster: arn:aws:eks:ap-northeast-3:123123123:cluster/yjkim-eks
    user: testuser
  name: testuser
current-context: testuser
kind: Config
preferences: {}
users:
...
- name: testuser
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - --region
      - ap-northeast-3
      - eks
      - get-token
      - --cluster-name
      - yjkim-eks
      - --output
      - json
      command: aws

(testuser:N/A) [root@yjkim-eks-bastion ~]# kubectl get node -v6
I0604 11:43:44.399383    8530 loader.go:374] Config loaded from file:  /root/.kube/config
I0604 11:43:45.177588    8530 round_trippers.go:553] GET https://45833AC6216CC4B071830959E68507F8.gr7.ap-northeast-3.eks.amazonaws.com/api/v1/nodes?limit=500 200 OK in 770 milliseconds
NAME                                               STATUS   ROLES    AGE   VERSION
ip-192-168-1-70.ap-northeast-3.compute.internal    Ready    <none>   17h   v1.24.13-eks-0a21954
ip-192-168-2-86.ap-northeast-3.compute.internal    Ready    <none>   17h   v1.24.13-eks-0a21954
ip-192-168-3-249.ap-northeast-3.compute.internal   Ready    <none>   17h   v1.24.13-eks-0a21954

# 방안2 : 아래 edit로 mapUsers 내용 직접 수정 system:authenticated
kubectl edit cm -n kube-system aws-auth

(testuser:default) [root@yjkim-eks-bastion ~]# eksctl get iamidentitymapping --cluster $CLUSTER_NAME
ARN												USERNAME				GROUPS					ACCOUNT
arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-ng1-NodeInstanceRole-1J7PRS1TBQQML	system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes
arn:aws:iam::123123123:user/testuser								testuser				system:authenticated

# 삭제 하면 적용 된다. 
(testuser:default) [root@yjkim-eks-bastion ~]# eksctl delete iamidentitymapping --cluster $CLUSTER_NAME --arn  arn:aws:iam::$ACCOUNT_ID:user/testuser
2023-06-04 11:52:37 [ℹ]  removing identity "arn:aws:iam::123123123:user/testuser" from auth ConfigMap (username = "testuser", groups = ["system:authenticated"])

(testuser:default) [root@yjkim-eks-bastion ~]# eksctl get iamidentitymapping --cluster $CLUSTER_NAME
ARN												USERNAME				GROUPS					ACCOUNT
arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-ng1-NodeInstanceRole-1J7PRS1TBQQML	system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl get cm -n kube-system aws-auth -o yaml | yh
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-ng1-NodeInstanceRole-1J7PRS1TBQQML
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    []
kind: ConfigMap
metadata:
  creationTimestamp: "2023-06-03T09:32:33Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "200738"
  uid: e8721b62-b971-4b9c-9d6d-3ce0def97e89

```

### IRSA 

* 동작 : K8s Pod -> AWS 서비스 이용시 -> AWS STS/IAM <-> IAM OIDC IdP 인증/인가

![IRSA1](/posts/study/cloudnet-aews/6w/images/IRSA1.png)
![IRSA2](/posts/study/cloudnet-aews/6w/images/IRSA2.png)

* 기본적으로 사용되는 EKS 의 pod IAM 정보들은 node 의 EC2 instance profile 을 따라가는것으로 알고 있으며 
* 최소 권한에 위배 되기 때문에 IRSA 를 사용 하도록 권장 하는것으로 알고 있다. 
* sample pod 의 automountServiceAccountToken 을 false 로 작성후 aws sdk 의 쿼리 해보면 node role 로 전송 하는것을 확인 할 수 있으며 
* sample pod 로 생성된 token 은 jwt token 으로 확인시에 eks iss 속성을 확인 할 수 있다. 
* eksctl 을 통해 iamserviceaccount 를 생성 한 후에 해당 값을 sa 로 구성 한다면 mutatingwebhook 를 이용하여 aws token 정보를 pod 에 구성 해준다. 

```sh 
eksctl create iamserviceaccount \
  --name my-sa \
  --namespace default \
  --cluster $CLUSTER_NAME \
  --approve \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn' --output text)

(testuser:default) [root@yjkim-eks-bastion ~]# eksctl get iamserviceaccount --cluster $CLUSTER_NAME
NAMESPACE	NAME				ROLE ARN
default		my-sa				arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-def-Role1-DQB2ABRNOM44
kube-system	aws-load-balancer-controller	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-C0NFUPNGFCSW

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl get sa
NAME      SECRETS   AGE
default   0         18h
my-sa     0         37s

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl describe sa my-sa
Name:                my-sa
Namespace:           default
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-def-Role1-DQB2ABRNOM44
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl get pod eks-iam-test3
NAME            READY   STATUS    RESTARTS   AGE
eks-iam-test3   1/1     Running   0          18s

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl describe pod eks-iam-test3
Name:             eks-iam-test3
...
Containers:
  my-aws-cli:
    Container ID:  containerd://605b8f708878291e31e3c74198bcc8949ba9c02b1e754c23782ba68996be2dfb
    Image:         amazon/aws-cli:latest
    Environment:
      AWS_STS_REGIONAL_ENDPOINTS:   regional
      AWS_DEFAULT_REGION:           ap-northeast-3
      AWS_REGION:                   ap-northeast-3
      AWS_ROLE_ARN:                 arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-def-Role1-DQB2ABRNOM44
      AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token


(testuser:default) [root@yjkim-eks-bastion ~]# eksctl get iamserviceaccount --cluster $CLUSTER_NAME
NAMESPACE	NAME				ROLE ARN
default		my-sa				arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-def-Role1-DQB2ABRNOM44
kube-system	aws-load-balancer-controller	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-C0NFUPNGFCSW

(testuser:default) [root@yjkim-eks-bastion ~]# kubectl exec -it eks-iam-test3 -- aws sts get-caller-identity --query Arn
"arn:aws:sts::123123123:assumed-role/eksctl-yjkim-eks-addon-iamserviceaccount-def-Role1-DQB2ABRNOM44/botocore-session-1685852496"

```

* 아래 내용에서는 k8s 의 aws-auth 를 변경 불가능하게 immutable 속성 지정 
* immutable 속성이 있는 cm 은 삭제후 재생성 해주어야 한다고 한다.