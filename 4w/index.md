---
title: "EKS Observability "
date: 2023-05-20T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 4w
    identifier: 4w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 


Cloudnet 에서 진행하는 AEWS 스터디 4주차 자료를 가지고 하는 블로깅이며 
이번 주차에서 다룰 내용은 모니터링을 포함한 Observability 이다. 

로깅 + 모니터링 + tracing 을 EKS 에서 어떻게 하는지?

혹은 AWS 에서 어떤 제품을 이용하여 수행 할 수 있는지에 대한 내용이 주가 되는 내용이다 
역시 PKOS 에서 중복되는 내용들이 있기 떄문에 중복되는 부분은 SKIP 할 듯하다. 



---

## 사전준비 

```sh 
# download cloudformation and deploy 
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick3.yaml

aws cloudformation deploy --template-file ./eks-oneclick3.yaml \
    --stack-name yjkim-eks --parameter-overrides KeyName=lala-yjkim SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32 MyIamUserAccessKeyID=... MyIamUserSecretAccessKey='....' ClusterBaseName=yjkim-eks \
    --region ap-northeast-3

ssh -i ~/.ssh/yjkim/lala-yjkim.pem ec2-user@$(aws cloudformation describe-stacks --region ap-northeast-3 --stack-name yjkim-eks --query 'Stacks[*].Outputs[0].OutputValue' --output text)


# node env setting 
N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3b -o jsonpath={.items[0].status.addresses[0].address})
N3=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-3c -o jsonpath={.items[0].status.addresses[0].address})
echo "export N1=$N1" >> /etc/profile
echo "export N2=$N2" >> /etc/profile
echo "export N3=$N3" >> /etc/profile
echo $N1, $N2, $N3

# security group  
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values='*ng1*' --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32


# external dns 
MyDomain=mydomain.link
echo "export MyDomain=mydomain.link" >> /etc/profile
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -

# aws lb controller 
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

# gp3 sc 
kubectl get sc
cat <<EOT > gp3-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3
allowVolumeExpansion: true
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  allowAutoIOPSPerGBIncrease: 'true'
  encrypted: 'true'
EOT
kubectl apply -f gp3-sc.yaml
kubectl get sc


# EFS csi driver 설치
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa

# EFS 스토리지클래스 생성 및 확인
curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
sed -i "s/fs-92107410/$EfsFsId/g" storageclass.yaml
kubectl apply -f storageclass.yaml
kubectl get sc efs-sc
```

---

## Logging

### EKS Controller Plain Logging

* aws cli 를 이용하여 EKS Controller plain 의 loging 을 활성화 할 수 있다. 
  * 각 컴포넌트별 명칭이 정의 되어있으며 아래 script 를 참고 하면 되겠다. 

```sh
# 로그 활성화 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws eks update-cluster-config --region $AWS_DEFAULT_REGION --name $CLUSTER_NAME \
>     --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

{
    "update": {
        "id": "5164c03d-eb2d-4595-9c6b-817e6b3352e0",
        "status": "InProgress",
        "type": "LoggingUpdate",
        "params": [
            {
                "type": "ClusterLogging",
                "value": "{\"clusterLogging\":[{\"types\":[\"api\",\"audit\",\"authenticator\",\"controllerManager\",\"scheduler\"],\"enabled\":true}]}"
            }
        ],
        "createdAt": "2023-05-20T14:26:42.505000+09:00",
        "errors": []
    }
}

# Cloudwatch 로그 그룹 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws logs describe-log-groups | jq

{
  "logGroups": [
    {
      "logGroupName": "/aws/eks/yjkim-eks/cluster",
      "creationTime": 1684560417701,
      "metricFilterCount": 0,
      "arn": "arn:aws:logs:ap-northeast-3:123123123:log-group:/aws/eks/yjkim-eks/cluster:*",
      "storedBytes": 0
    }
  ]
}

# 로그 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws logs tail /aws/eks/$CLUSTER_NAME/cluster | more
2023-05-20T05:24:43.000000+00:00 cloud-controller-manager-900118300c7497e54b1f844e21ac4f7b I0520 05:24:43.888087      11 tagging_controller.go:15
2] Skip putting node ip-192-168-3-75.ap-northeast-3.compute.internal in work queue since it was already tagged earlier.
2023-05-20T05:26:57.511000+00:00 kube-apiserver-audit-900118300c7497e54b1f844e21ac4f7b {"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Re
quest","auditID":"9151d8b4-0630-4020-8e18-c65369a13249","stage":"ResponseComplete","requestURI":"/apis/policy/v1/poddisruptionbudgets?limit=500\u
0026resourceVersion=0","verb":"list","user":{"username":"system:kube-scheduler","groups":["system:authenticated"]},"sourceIPs":["10.0.60.42"],"us
erAgent":"kube-scheduler/v1.24.13 (linux/amd64) kubernetes/6305d65/scheduler","objectRef":{"resource":"poddisruptionbudgets","apiGroup":"policy",
"apiVersion":"v1"},"responseStatus":{"metadata":{},"status":"Failure","message":"poddisruptionbudgets.policy is forbidden: User \"system:kube-sch
eduler\" cannot list resource \"poddisruptionbudgets\" in API group \"policy\" at the cluster scope","reason":"Forbidden","details":{"group":"pol
icy","kind":"poddisruptionbudgets"},"code":403},"requestReceivedTimestamp":"2023-05-20T04:14:53.795931Z","stageTimestamp":"2023-05-20T04:14:53.79
8812Z","annotations":{"authorization.k8s.io/decision":"forbid","authorization.k8s.io/reason":""}}

## 아래 예시로 출력 옵션을 정의 할 수 있다. 

# 신규 로그를 바로 출력
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --follow

# 필터 패턴
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --filter-pattern <필터 패턴>

# 로그 스트림이름
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --log-stream-name-prefix <로그 스트림 prefix> --follow
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --log-stream-name-prefix kube-controller-manager --follow
kubectl scale deployment -n kube-system coredns --replicas=1
kubectl scale deployment -n kube-system coredns --replicas=2

# 시간 지정: 1초(s) 1분(m) 1시간(h) 하루(d) 한주(w)
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --since 1h30m

# 짧게 출력
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --since 1h30m --format short


# cloudwatch 에서 PPL 포멧으로 쿼리 
aws logs get-query-results --query-id $(aws logs start-query \
--log-group-name '/aws/eks/yjkim-eks/cluster' \
--start-time `date -d "-1 hours" +%s` \
--end-time `date +%s` \
--query-string 'fields @timestamp, @message | filter @logStream ~= "kube-controller-manager" | sort @timestamp desc' \
| jq --raw-output '.queryId')

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws logs get-query-results --query-id $(aws logs start-query \
>   --log-group-name '/aws/eks/yjkim-eks/cluster' \
>   --start-time `date -d "-1 hours" +%s` \
>   --end-time `date +%s` \
>   --query-string 'fields @timestamp, @message | filter @logStream ~= "kube-scheduler" | sort @timestamp desc' \
>   | jq --raw-output '.queryId')
{
    "results": [],
    "statistics": {
        "recordsMatched": 0.0,
        "recordsScanned": 0.0,
        "bytesScanned": 0.0
    },
    "status": "Running"
}


# 로깅 비활성화 
eksctl utils update-cluster-logging --cluster $CLUSTER_NAME --region $AWS_DEFAULT_REGION --disable-types all --approve

# 로그 그룹 삭제 
aws logs delete-log-group --log-group-name /aws/eks/$CLUSTER_NAME/cluster
```

* Controller Plain 의 로그중에는 중요한 몇가지의 로그를 모니터링 할 수 있는 예시를 제공 한다고 한다. 
  * 참고 링크 : https://aws.amazon.com/ko/blogs/containers/managing-etcd-database-size-on-amazon-eks-clusters/


```sh 
# etcd 의 사이즈를 kubectl 로 조회 할 수 있다.
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get --raw /metrics | grep "etcd_db_total_size_in_bytes"
# HELP etcd_db_total_size_in_bytes [ALPHA] Total size of the etcd database file physically allocated in bytes.
# TYPE etcd_db_total_size_in_bytes gauge
etcd_db_total_size_in_bytes{endpoint="http://10.0.160.16:2379"} 4.624384e+06
etcd_db_total_size_in_bytes{endpoint="http://10.0.32.16:2379"} 4.624384e+06
etcd_db_total_size_in_bytes{endpoint="http://10.0.96.16:2379"} 4.714496e+06

# 해당 값이 임계치를 도달하면 api-server 에서 에러로그를 출력해주는데 Cloudwatch 에서도 아래와 같이 확인 해볼 수 있다. 
fields @timestamp, @message, @logStream
| filter @logStream like /kube-apiserver-audit/
| filter @message like /mvcc: database space exceeded/
| limit 10

# 저장 되어있는 값들이 많아서 용량을 초과 했다면 어떠한 것들이 초과 했는지에 대한 확인 방법은 아래와 같다. 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get --raw=/metrics | grep apiserver_storage_objects |awk '$2>50' |sort -g -k 2
# HELP apiserver_storage_objects [STABLE] Number of stored objects at the time of last check split by kind.
# TYPE apiserver_storage_objects gauge
apiserver_storage_objects{resource="clusterrolebindings.rbac.authorization.k8s.io"} 75
apiserver_storage_objects{resource="clusterroles.rbac.authorization.k8s.io"} 89
apiserver_storage_objects{resource="events"} 94
# resource 의 객체가 몇개를 저장하고 있는것을 출력해준다. 

# 아래 몇가지 예시들을 추가로 참고 링크에서 언급 하고 있다. 
# CW Logs Insights 쿼리 : Request volume - Requests by User Agent:
fields userAgent, requestURI, @timestamp, @message
| filter @logStream like /**kube-apiserver-audit**/
| stats count(*) as count by userAgent
| sort count desc

# CW Logs Insights 쿼리 : Request volume - Requests by Universal Resource Identifier (URI)/Verb:
filter @logStream like /**kube-apiserver-audit**/
| stats count(*) as count by requestURI, verb, user.username
| sort count desc

# Object revision updates
fields requestURI
| filter @logStream like /**kube-apiserver-audit**/
| filter requestURI like /pods/
| filter verb like /patch/
| filter count > 8
| stats count(*) as count by requestURI, responseStatus.code
| filter responseStatus.code not like /500/
| sort count desc

#
fields @timestamp, userAgent, responseStatus.code, requestURI
| filter @logStream like /**kube-apiserver-audit**/
| filter requestURI like /pods/
| filter verb like /patch/
| filter requestURI like /name_of_the_pod_that_is_updating_fast/
```


### Container Logging

* Container 의 경우 /dev/stdout, /dev/stderr 로 로깅을 전달 하는것을 권고 하고 있다. 
* kubectl logs 단일 명령으로 조회 할 수 있도록 하기 위함으로 보인다. 

```sh 
# nginx 배포 

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# CERT_ARN=$(aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn[]' --output text)

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo $CERT_ARN
arn:aws:acm:ap-northeast-3:123123123:certificate/1d6f53d5-518b-49e7-836d-299d334a3eea
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo $MyDomain
oct28-yjkim.ml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat <<EOT > nginx-values.yaml
> service:
>     type: NodePort
>
> ingress:
>   enabled: true
>   ingressClassName: alb
>   hostname: nginx.$MyDomain
>   path: /*
>   annotations:
>     alb.ingress.kubernetes.io/scheme: internet-facing
>     alb.ingress.kubernetes.io/target-type: ip
>     alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
>     alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
>     alb.ingress.kubernetes.io/success-codes: 200-399
>     alb.ingress.kubernetes.io/load-balancer-name: $CLUSTER_NAME-ingress-alb
>     alb.ingress.kubernetes.io/group.name: study
>     alb.ingress.kubernetes.io/ssl-redirect: '443'
> EOT
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat nginx-values.yaml | yh

service:
    type: NodePort
ingress:
  enabled: true
  ingressClassName: alb
  hostname: nginx.oct28-yjkim.ml
  path: /*
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-3:123123123:certificate/1d6f53d5-518b-49e7-836d-299d334a3eea
    alb.ingress.kubernetes.io/success-codes: 200-399
    alb.ingress.kubernetes.io/load-balancer-name: yjkim-eks-ingress-alb
    alb.ingress.kubernetes.io/group.name: study
    alb.ingress.kubernetes.io/ssl-redirect: '443'
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]#
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm install nginx bitnami/nginx --version 14.1.0 -f nginx-values.yaml
NAME: nginx
LAST DEPLOYED: Sat May 20 14:51:00 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 14.1.0
APP VERSION: 1.24.0

** Please be patient while the chart is being deployed **
NGINX can be accessed through the following DNS name from within your cluster:

    nginx.default.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL and associate its hostname to your cluster external IP:

   export CLUSTER_IP=$(minikube ip) # On Minikube. Use: `kubectl cluster-info` on others K8s clusters
   echo "NGINX URL: http://nginx.oct28-yjkim.ml"
   echo "$CLUSTER_IP  nginx.oct28-yjkim.ml" | sudo tee -a /etc/hosts
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get ingress,deploy,svc,ep nginx

NAME                              CLASS   HOSTS            ADDRESS                                                            PORTS   AGE
ingress.networking.k8s.io/nginx   alb     nginx.oct28-yjkim.ml   yjkim-eks-ingress-alb-848530280.ap-northeast-3.elb.amazonaws.com   80      7s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   0/1     1            0           7s

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/nginx   NodePort   10.100.94.249   <none>        80:32255/TCP   7s

NAME              ENDPOINTS   AGE
endpoints/nginx               7s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get targetgroupbindings # ALB TG 확인

NAME                           SERVICE-NAME   SERVICE-PORT   TARGET-TYPE   AGE
k8s-default-nginx-6a99308b99   nginx          http           ip            5s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]#
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo -e "Nginx WebServer URL = https://nginx.$MyDomain"
Nginx WebServer URL = https://nginx.oct28-yjkim.ml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s https://nginx.$MyDomain
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl logs deploy/nginx -f
nginx 05:51:07.56
nginx 05:51:07.56 Welcome to the Bitnami nginx container
nginx 05:51:07.56 Subscribe to project updates by watching https://github.com/bitnami/containers
nginx 05:51:07.56 Submit issues and feature requests at https://github.com/bitnami/containers/issues
nginx 05:51:07.56
nginx 05:51:07.57 INFO  ==> ** Starting NGINX setup **
nginx 05:51:07.58 INFO  ==> Validating settings in NGINX_* env vars
Generating RSA private key, 4096 bit long modulus (2 primes)
...........................................................................................................................................................................................++++
................................................................++++
e is 65537 (0x010001)
Signature ok
subject=CN = example.com
Getting Private key
nginx 05:51:08.57 INFO  ==> No custom scripts in /docker-entrypoint-initdb.d
nginx 05:51:08.57 INFO  ==> Initializing NGINX
realpath: /bitnami/nginx/conf/vhosts: No such file or directory
nginx 05:51:08.59 INFO  ==> ** NGINX setup finished! **

nginx 05:51:08.60 INFO  ==> ** Starting NGINX **

# 반복 접속 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# while true; do curl -s https://nginx.$MyDomain -I | head -n 1; date; sleep 1; done
Sat May 20 14:53:37 KST 2023
Sat May 20 14:53:38 KST 2023
Sat May 20 14:53:39 KST 2023
Sat May 20 14:53:40 KST 2023
^C


# 컨테이너 로그 파일 위치 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl exec -it deploy/nginx -- ls -l /opt/bitnami/nginx/logs/
total 0
lrwxrwxrwx 1 root root 11 Apr 24 10:13 access.log -> /dev/stdout
lrwxrwxrwx 1 root root 11 Apr 24 10:13 error.log -> /dev/stderr

```

* 유의사항으로는 종료된 파드의 로그는 kubectl logs 로 조회 할 수 없으며 kubelet 의 기본 설정으로 로그 파일의 최대 크기가 10Mi 를 초괴 하는 로그는 조회 할수 없다. 
* 아래는 정리가 많이 안된 글이기는 하지만 managed 의 경우 기존 kubelet config + log 변경하기 위한 설정은 eks managedgroup 에서 변경이 가능 하다. 

```sh 
ssh ec2-user@$N1 cat /etc/kubernetes/kubelet/kubelet-config.json  | grep Log
...
containerLogMaxSize: 10Mi

# eks nodegroup 에 적용 되어있는 log 설정 확인 
[ec2-user@ip-192-168-1-97 ~]$ kubelet --help | grep container| grep log
      --container-log-max-files int32                            <Warning: Beta feature> Set the maximum number of container log files that can be present for a container. The number must be >= 2. This flag can only be used with --container-runtime=remote. (default 5) (DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.)
      --container-log-max-size string                            <Warning: Beta feature> Set the maximum size (e.g. 10Mi) of container log file before it is rotated. This flag can only be used with --container-runtime=remote. (default "10Mi") (DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.)

[ec2-user@ip-192-168-1-97 ~]$ systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubelet-args.conf, 30-kubelet-extra-args.conf

[ec2-user@ip-192-168-1-97 ~]$ cat /etc/systemd/system/kubelet.service.d/10-kubelet-args.conf
[Service]
Environment='KUBELET_ARGS=--node-ip=192.168.1.97 --pod-infra-container-image=602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.5 --v=2 --cloud-provider=aws --container-runtime=remote'
[ec2-user@ip-192-168-1-97 ~]$ cat /etc/systemd/system/kubelet.service.d/30-kubelet-extra-args.conf
[Service]
Environment='KUBELET_EXTRA_ARGS=--node-labels=eks.amazonaws.com/sourceLaunchTemplateVersion=1,alpha.eksctl.io/cluster-name=yjkim-eks,alpha.eksctl.io/nodegroup-name=ng1,eks.amazonaws.com/nodegroup-image=ami-01bb9ccf4f71beebd,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=ng1,eks.amazonaws.com/sourceLaunchTemplateId=lt-0aca55bca32166619 --max-pods=58 --max-pods=50'

[ec2-user@ip-192-168-1-97 ~]$ cat /etc/kubernetes/kubelet/kubelet-config.json  | grep log

# 없다. 기본으로 사용중인듯 
# node group 에 설정 변경후 적용 되는지 확인 해보자 

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cp myeks.yaml myeks-largelog.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat myeks-largelog.yaml  | grep kubeletEx -a2
  volumeThroughput: 125
  volumeType: gp3
  kubeletExtraConfig:
    containerLogMaxSize: "500Mi"
    containerLogMaxFiles: 5

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get cluster
NAME		REGION		EKSCTL CREATED
yjkim-eks	ap-northeast-3	True

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get nodegroup --cluster yjkim-eks
CLUSTER		NODEGROUP	STATUS	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE IDASG NAME					TYPE
yjkim-eks	ng1		ACTIVE	2023-05-20T04:23:39Z	3

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl update nodegroup -f myeks-largelog.yaml
Error: loading config file "myeks-largelog.yaml": error unmarshaling JSON: while decoding JSON: json: unknown field "kubeletExtraConfig"

# eksctl schema 를 확인 해보니 nodegroup 에는 있는데 managedNodegroup 에는 없다. 
## https://eksctl.io/usage/schema/#nodeGroups-kubeletExtraConfig
# eks nodegroup 에서 bootstrap.sh 에 인수 전달 하기 위한 방법을 제공 해준다.
## eks > userguide > launch-templates
## https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/launch-templates.html

# certificate-authority 
EKS_CA=$(aws eks describe-cluster --query "cluster.certificateAuthority.data" --output text --name yjkim-eks --region ap-northeast-3)

# api-server-endpoint
EKS_API_ENDPOINT=$(aws eks describe-cluster --query "cluster.endpoint" --output text --name yjkim-eks  --region ap-northeast-3)

# --dns-cluster-ip 
EKS_DNS_IP=$(aws eks describe-cluster --query "cluster.kubernetesNetworkConfig.serviceIpv4Cidr" --output text --name yjkim-eks --region ap-northeast-3)

# managed nodegroup 에 옵션 추가 
    overrideBootstrapCommand: |
      #!/bin/bash
      /etc/eks/bootstrap.sh yjkim-eks \
        --b64-cluster-ca 'LS0tLS1CRUdJTiB...' \
        --apiserver-endpoint https://1BB88B99212123.yl4.ap-northeast-3.eks.amazonaws.com \
        --dns-cluster-ip 10.100.0.10 \
        --container-runtime containerd \
        --kubelet-extra-args '--container-log-max-size=500Mi' \
        --kubelet-extra-args '--container-log-max-files=6'

# 재시도 
eksctl update nodegroup -f myeks-largelog.yaml
2023-05-20 15:46:00 [ℹ]  validating nodegroup "ng1"
2023-05-20 15:46:00 [!]  unchanged fields for nodegroup ng1: the following fields remain unchanged; they are not supported by `eksctl update nodegroup`: AMIFamily, InstanceType, ScalingConfig, VolumeSize, SSH, Labels, Tags, IAM, SecurityGroups, MaxPodsPerNode, VolumeType, VolumeIOPS, VolumeThroughput, PreBootstrapCommands, OverrideBootstrapCommand, DisableIMDSv1, DisablePodIMDS, InstanceSelector
Error: managedNodeGroups[0].overrideBootstrapCommand can only be set when a custom AMI (managedNodeGroups[0].ami) is specified

# 있는 애를 업데이트 하지말고 새로 생성하고 log 옵션이 적용 되는걸 확인 해보자. 
# 지난주 ng 생성 참고 
eksctl create nodegroup --help

# 해당 리전의 ami 찾기 
ami-01bb9ccf4f71beebd

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl create nodegroup --help | grep ami
      --node-ami string                'auto-ssm', 'auto' or an AMI ID (advanced use)

eksctl create nodegroup -c $CLUSTER_NAME -r $AWS_DEFAULT_REGION --subnet-ids "$PubSubnet1","$PubSubnet2","$PubSubnet3" --ssh-access \
  -n ng2 -t c5d.large -N 1 -m 1 -M 1 --node-volume-size=30 --node-labels disk=nvme--dry-run > nodegroup2.yaml

cat << EOF > nodegroup2.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: yjkim-eks
  region: ap-northeast-3

managedNodeGroups:
  - name: my-nodegroup
    ami: ami-01bb9ccf4f71beebd
    instanceType: m5.large
    privateNetworking: true
    disableIMDSv1: true
    labels: { x86-al2-specified-mng }
    overrideBootstrapCommand: |
      #!/bin/bash
      /etc/eks/bootstrap.sh my-cluster \
        --b64-cluster-ca certificate-authority \
        --apiserver-endpoint api-server-endpoint \
        --dns-cluster-ip service-cidr.10 \
        --container-runtime containerd \
        --kubelet-extra-args '--max-pods=my-max-pods-value' \
        --use-max-pods false

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat myng2.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
managedNodeGroups:
- ami: ami-01bb9ccf4f71beebd
  desiredCapacity: 1
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
  instanceType: c5d.large
  labels:
    alpha.eksctl.io/cluster-name: yjkim-eks
    alpha.eksctl.io/nodegroup-name: ng2
    disk: nvme
  maxPodsPerNode: 100
  maxSize: 1
  minSize: 1
  name: ng2
  privateNetworking: false
  releaseVersion: ""
  securityGroups:
    withLocal: null
    withShared: null
  ssh:
    allow: true
    publicKeyPath: ~/.ssh/id_rsa.pub
  subnets:
  - subnet-05518687a5a7a6d5b
  - subnet-0888d6881dbe54024
  - subnet-0c67132e3042cf8cb
  tags:
    alpha.eksctl.io/nodegroup-name: ng2
    alpha.eksctl.io/nodegroup-type: managed
  volumeIOPS: 3000
  volumeSize: 30
  volumeThroughput: 125
  volumeType: gp3
  overrideBootstrapCommand: |
    #!/bin/bash
    /etc/eks/bootstrap.sh yjkim-eks \
      --b64-cluster-ca 'LS0tLS1CRUdJTiB.....' \
      --apiserver-endpoint 'https://1BB88B99259......yl4.ap-northeast-3.eks.amazonaws.com' \
      --dns-cluster-ip 10.100.0.10 \
      --container-runtime containerd \
      --kubelet-extra-args '--container-log-max-size=500Mi --container-log-max-files=6 ' 
metadata:
  name: yjkim-eks
  region: ap-northeast-3
  version: "1.24"
EOF 

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl create nodegroup -f nodegroup2.yaml


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get no
NAME                                              STATUS   ROLES    AGE     VERSION
ip-192-168-1-39.ap-northeast-3.compute.internal   Ready    <none>   9m10s   v1.24.13-eks-0a21954
ip-192-168-1-97.ap-northeast-3.compute.internal   Ready    <none>   3h37m   v1.24.13-eks-0a21954
ip-192-168-2-26.ap-northeast-3.compute.internal   Ready    <none>   3h37m   v1.24.13-eks-0a21954
ip-192-168-3-75.ap-northeast-3.compute.internal   Ready    <none>   3h37m   v1.24.13-eks-0a21954

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# ssh ec2-user@192.168.1.39

# 아래 kubelet process 를 보면 옵션 변경이 적용 된것을 확인 할 수있다. 
# 물론 운영중인 클러스터라면 기존에 설정된 extra config 를 수동으로 포함 해주어서 맞는 값인지는 확인 해주어야 한다. 
[ec2-user@ip-192-168-1-39 ~]$ ps -ef | grep kubelet
root      2875     1  1 08:11 ?        00:00:02 /usr/bin/kubelet --config /etc/kubernetes/kubelet/kubelet-config.json --kubeconfig /var/lib/kubelet/kubeconfig --container-runtime-endpoint unix:///run/containerd/containerd.sock --image-credential-provider-config /etc/eks/image-credential-provider/config.json --image-credential-provider-bin-dir /etc/eks/image-credential-provider --node-ip=192.168.2.88 --pod-infra-container-image=602401143452.dkr.ecr.ap-northeast-3.amazonaws.com/eks/pause:3.5 --v=2 --cloud-provider=aws --container-runtime=remote --container-log-max-size=500Mi --container-log-max-files=6
root      3638  3052  0 07:53 ?        00:00:00 /csi-node-driver-registrar --csi-address=/csi/csi.sock --kubelet-registration-path=/var/lib/kubelet/plugins/efs.csi.aws.com/csi.sock --v=2
root      3952  3832  0 07:53 ?        00:00:00 /csi-node-driver-registrar --csi-address=/csi/csi.sock --kubelet-registration-path=/var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock --v=2
ec2-user  7241  7137  0 08:01 pts/0    00:00:00 grep --color=auto kubelet

```

## Container Insight metrics in Cloudwatch & Fluent bit 

```sh 
FluentBitHttpServer='On'
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
FluentBitReadFromTail='On'
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_DEFAULT_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_DEFAULT_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -

namespace/amazon-cloudwatch created
serviceaccount/cloudwatch-agent created
clusterrole.rbac.authorization.k8s.io/cloudwatch-agent-role created
clusterrolebinding.rbac.authorization.k8s.io/cloudwatch-agent-role-binding created
configmap/cwagentconfig created
daemonset.apps/cloudwatch-agent created
configmap/fluent-bit-cluster-info created
serviceaccount/fluent-bit created
clusterrole.rbac.authorization.k8s.io/fluent-bit-role created
clusterrolebinding.rbac.authorization.k8s.io/fluent-bit-role-binding created
configmap/fluent-bit-config created
daemonset.apps/fluent-bit created
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]#
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get-all -n amazon-cloudwatch
NAME                                                NAMESPACE          AGE
configmap/cwagent-clusterleader                     amazon-cloudwatch  1s
configmap/cwagentconfig                             amazon-cloudwatch  10s
configmap/fluent-bit-cluster-info                   amazon-cloudwatch  10s
configmap/fluent-bit-config                         amazon-cloudwatch  10s
configmap/kube-root-ca.crt                          amazon-cloudwatch  10s
pod/cloudwatch-agent-2h2q5                          amazon-cloudwatch  10s
pod/cloudwatch-agent-tjq8m                          amazon-cloudwatch  10s
pod/cloudwatch-agent-tqnnk                          amazon-cloudwatch  10s
pod/cloudwatch-agent-zv8lf                          amazon-cloudwatch  10s
pod/fluent-bit-2g846                                amazon-cloudwatch  10s
pod/fluent-bit-dlwpw                                amazon-cloudwatch  10s
pod/fluent-bit-fw6ft                                amazon-cloudwatch  10s
pod/fluent-bit-jpfgg                                amazon-cloudwatch  10s
serviceaccount/cloudwatch-agent                     amazon-cloudwatch  10s
serviceaccount/default                              amazon-cloudwatch  10s
serviceaccount/fluent-bit                           amazon-cloudwatch  10s
controllerrevision.apps/cloudwatch-agent-bfcddf455  amazon-cloudwatch  10s
controllerrevision.apps/fluent-bit-98d767484        amazon-cloudwatch  10s
daemonset.apps/cloudwatch-agent                     amazon-cloudwatch  10s
daemonset.apps/fluent-bit                           amazon-cloudwatch  10s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get ds,pod,cm,sa -n amazon-cloudwatch
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/cloudwatch-agent   4         4         4       4            4           kubernetes.io/os=linux   12s
daemonset.apps/fluent-bit         4         4         4       4            4           <none>                   12s

NAME                         READY   STATUS    RESTARTS   AGE
pod/cloudwatch-agent-2h2q5   1/1     Running   0          12s
pod/cloudwatch-agent-tjq8m   1/1     Running   0          12s
pod/cloudwatch-agent-tqnnk   1/1     Running   0          12s
pod/cloudwatch-agent-zv8lf   1/1     Running   0          12s
pod/fluent-bit-2g846         1/1     Running   0          12s
pod/fluent-bit-dlwpw         1/1     Running   0          12s
pod/fluent-bit-fw6ft         1/1     Running   0          12s
pod/fluent-bit-jpfgg         1/1     Running   0          12s

NAME                                DATA   AGE
configmap/cwagent-clusterleader     0      3s
configmap/cwagentconfig             1      12s
configmap/fluent-bit-cluster-info   6      12s
configmap/fluent-bit-config         5      12s
configmap/kube-root-ca.crt          1      12s

NAME                              SECRETS   AGE
serviceaccount/cloudwatch-agent   0         12s
serviceaccount/default            0         12s
serviceaccount/fluent-bit         0         12s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe clusterrole cloudwatch-agent-role fluent-bit-role                          # 클러스터롤 확인
Name:         cloudwatch-agent-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names           Verbs
  ---------         -----------------  --------------           -----
  configmaps        []                 []                       [create]
  events            []                 []                       [create]
  nodes/stats       []                 []                       [create]
  configmaps        []                 [cwagent-clusterleader]  [get update]
  nodes/proxy       []                 []                       [get]
  endpoints         []                 []                       [list watch]
  nodes             []                 []                       [list watch]
  pods              []                 []                       [list watch]
  replicasets.apps  []                 []                       [list watch]
  jobs.batch        []                 []                       [list watch]


Name:         fluent-bit-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources    Non-Resource URLs  Resource Names  Verbs
  ---------    -----------------  --------------  -----
  namespaces   []                 []              [get list watch]
  nodes/proxy  []                 []              [get list watch]
  nodes        []                 []              [get list watch]
  pods/logs    []                 []              [get list watch]
  pods         []                 []              [get list watch]
               [/metrics]         []              [get]
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe clusterrolebindings cloudwatch-agent-role-binding fluent-bit-role-binding  # 클러스터롤 바인딩 확인
Name:         cloudwatch-agent-role-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cloudwatch-agent-role
Subjects:
  Kind            Name              Namespace
  ----            ----              ---------
  ServiceAccount  cloudwatch-agent  amazon-cloudwatch


Name:         fluent-bit-role-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  fluent-bit-role
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  fluent-bit  amazon-cloudwatch


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe -n amazon-cloudwatch ds cloudwatch-agent
Name:           cloudwatch-agent
Selector:       name=cloudwatch-agent
Node-Selector:  kubernetes.io/os=linux
Labels:         <none>
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 4
Current Number of Nodes Scheduled: 4
Number of Nodes Scheduled with Up-to-date Pods: 4
Number of Nodes Scheduled with Available Pods: 4
Number of Nodes Misscheduled: 0
Pods Status:  4 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
    Mounts:
      /dev/disk from devdisk (ro)
      /etc/cwagentconfig from cwagentconfig (rw)
      /rootfs from rootfs (ro)
      /run/containerd/containerd.sock from containerdsock (ro)
      /sys from sys (ro)
      /var/lib/docker from varlibdocker (ro)
      /var/run/docker.sock from dockersock (ro)
  Volumes:
   cwagentconfig:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      cwagentconfig
    Optional:  false
   rootfs:
    Type:          HostPath (bare host directory volume)
    Path:          /
    HostPathType:
   dockersock:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/docker.sock
    HostPathType:
   varlibdocker:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker
    HostPathType:
   containerdsock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/containerd/containerd.sock
    HostPathType:
   sys:
    Type:          HostPath (bare host directory volume)
    Path:          /sys
    HostPathType:
   devdisk:
    Type:          HostPath (bare host directory volume)
    Path:          /dev/disk/
    HostPathType:
# / 경로를 풀로 공유 중

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get cm -n amazon-cloudwatch fluent-bit-cluster-info -o yaml | yh
apiVersion: v1
data:
  cluster.name: yjkim-eks
  http.port: "2020"
  http.server: "On"
  logs.region: ap-northeast-3
  read.head: "Off"
  read.tail: "On"

# fluent 에서 다른 곳도 모니러팅 하지만 기본적으로 아래 로그 등을 모니터링 하고 있다. 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe cm fluent-bit-config -n amazon-cloudwatch | grep Path
    Path                /var/log/journal
    Path                /var/log/containers/aws-node*, /var/log/containers/kube-proxy*
    Path                /var/log/dmesg
    Path                /var/log/messages
    Path                /var/log/secure
    Exclude_Path        /var/log/containers/cloudwatch-agent*, /var/log/containers/fluent-bit*, /var/log/containers/aws-node*, /var/log/containers/kube-proxy*
    Path                /var/log/containers/*.log
    Path                /var/log/containers/fluent-bit*
    Path                /var/log/containers/cloudwatch-agent*


# 이제 Cloudwatch -> Loggroup 에서 pod 의 로깅을 확인 가능 
# Cloudwatch -> Container insight pod metric 조회 가능 
# 우상단 맵에서 보기 클릭시 Container 맵 확인 가능 


```

![container-insight-log](/posts/study/cloudnet-aews/4w/images/container-insight-log.png)
![container-insight-map](/posts/study/cloudnet-aews/4w/images/container-insight-map.png)

## 삭제 

```sh 
eksctl delete cluster --name $CLUSTER_NAME && aws cloudformation delete-stack --stack-name $CLUSTER_NAME

# EKS Control Plane 로깅(CloudWatch Logs) 비활성화
eksctl utils update-cluster-logging --cluster $CLUSTER_NAME --region $AWS_DEFAULT_REGION --disable-types all --approve
# 로그 그룹 삭제 : 컨트롤 플레인
aws logs delete-log-group --log-group-name /aws/eks/$CLUSTER_NAME/cluster

---
# 로그 그룹 삭제 : 데이터 플레인
aws logs delete-log-group --log-group-name /aws/containerinsights/$CLUSTER_NAME/application
aws logs delete-log-group --log-group-name /aws/containerinsights/$CLUSTER_NAME/dataplane
aws logs delete-log-group --log-group-name /aws/containerinsights/$CLUSTER_NAME/host
aws logs delete-log-group --log-group-name /aws/containerinsights/$CLUSTER_NAME/performance
```