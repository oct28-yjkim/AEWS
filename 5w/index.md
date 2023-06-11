---
title: "EKS AutoScaling "
date: 2023-06-02T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 5w
    identifier: 5w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 


* Cloudnet 에서 진행하는 AEWS 스터디 5주차 AutoScaling 주제이다. 
* EKS 를 이용중에 서비스의 트레픽이 몰릴 경우의 확장에 대해서 이야기를 하고 있으며 
* 다른곳에서도 널리 사용중에 있는 스케일링 오픈소스 등을 언급 및 사용 할 예정이다. 
  * Horizontal Pod Autoscaler
  * Vertical Pod Autoscaler
  * Cluster Autoscaler 
  * Karpenter 
  * KEDA


### HPA, VPA

![hpa-vpa-overview](/posts/study/cloudnet-aews/5w/images/hpa-vpa-overview.png)

* 위 그림에서 보듯이 HPA 는 1개 pod 에 메트릭을 기반으로 모니터링 하다가 pod 를 증가, 감소 하도록 조절 해준다. 
* VPA 는 1개 Pod 의 resources 즉 CPU, MEMORY 등을 증가 시킨다. 

### CA

![ca-overview](/posts/study/cloudnet-aews/5w/images/ca-overview.png)

* Cluster Autoscaler 는 pending pods 를 모니터링 한 후에  스케줄링이 안된 포드가 있을시에 node 를 증설 한다. 

### karpenter

![karpenter-overview](/posts/study/cloudnet-aews/5w/images/karpenter-overview.png)

* karpenter 는 bin packing, 리소스를 모니터링 한후에 정리, 혹은 unschedulable pod 발생시 asg 관리를 받지 않고 증설 하는걸로 보인다.

---

## 사전준비 

```sh 
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick4.yaml

# CloudFormation 스택 배포
예시) aws cloudformation deploy --template-file eks-oneclick4.yaml --stack-name myeks --parameter-overrides KeyName=kp-gasida SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32  MyIamUserAccessKeyID=AKIA5... MyIamUserSecretAccessKey='CVNa2...' ClusterBaseName=myeks --region ap-northeast-2

# CloudFormation 스택 배포 완료 후 작업용 EC2 IP 출력
aws cloudformation describe-stacks --stack-name yjkim-eks --query 'Stacks[*].Outputs[0].OutputValue' --output text

# 작업용 EC2 SSH 접속
ssh -i ~/.ssh/yjkim/lala-yjkim.pem ec2-user@$(aws cloudformation describe-stacks --stack-name yjkim-eks --query 'Stacks[*].Outputs[0].OutputValue' --output text)
13.208.189.95

# default 네임스페이스 적용
kubectl ns default

# (옵션) context 이름 변경
NICK=<각자 자신의 닉네임>
NICK=gasida
kubectl ctx
kubectl config rename-context admin@myeks.ap-northeast-2.eksctl.io $NICK@myeks

# ExternalDNS
MyDomain=oct28-yjkim.ml
echo "export MyDomain=<자신의 도메인>" >> /etc/profile
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -

# kube-ops-view
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --set env.TZ="Asia/Seoul" --namespace kube-system
kubectl patch svc -n kube-system kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'
kubectl annotate service kube-ops-view -n kube-system "external-dns.alpha.kubernetes.io/hostname=kubeopsview.$MyDomain"
echo -e "Kube Ops View URL = http://kubeopsview.$MyDomain:8080/#scale=1.5"

# AWS LB Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

# 노드 보안그룹 ID 확인
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values='*ng1*' --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32


# 사용 리전의 인증서 ARN 확인
CERT_ARN=`aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn[]' --output text`
echo $CERT_ARN

arn:aws:acm:ap-northeast-3:123123123:certificate/1d6f53d5-518b-49e7-836d-299d334a3eea

# repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 파라미터 파일 생성
cat <<EOT > monitor-values.yaml
prometheus:
  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 5d
    retentionSize: "10GiB"

  verticalPodAutoscaler:
    enabled: true

  ingress:
    enabled: true
    ingressClassName: alb
    hosts: 
      - prometheus.$MyDomain
    paths: 
      - /*
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
      alb.ingress.kubernetes.io/group.name: study
      alb.ingress.kubernetes.io/ssl-redirect: '443'

grafana:
  defaultDashboardsTimezone: Asia/Seoul
  adminPassword: prom-operator

  ingress:
    enabled: true
    ingressClassName: alb
    hosts: 
      - grafana.$MyDomain
    paths: 
      - /*
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
      alb.ingress.kubernetes.io/group.name: study
      alb.ingress.kubernetes.io/ssl-redirect: '443'

defaultRules:
  create: false
kubeControllerManager:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
alertmanager:
  enabled: false
EOT

# 배포
kubectl create ns monitoring
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 45.27.2 \
--set prometheus.prometheusSpec.scrapeInterval='15s' --set prometheus.prometheusSpec.evaluationInterval='15s' \
-f monitor-values.yaml --namespace monitoring

# Metrics-server 배포
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# go 설치
yum install -y go

# EKS Node Viewer 설치 : 현재 ec2 spec에서는 설치에 다소 시간이 소요됨 = 2분 이상
go install github.com/awslabs/eks-node-viewer/cmd/eks-node-viewer@latest

# bin 확인 및 사용 
tree ~/go/bin
cd ~/go/bin
./eks-node-viewer
3 nodes (875m/5790m) 15.1% cpu ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ $0.156/hour | $113.880/month
20 pods (0 pending 20 running 20 bound)

ip-192-168-3-196.ap-northeast-2.compute.internal cpu ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░  22% (7 pods) t3.medium/$0.0520 On-Demand
ip-192-168-1-91.ap-northeast-2.compute.internal  cpu ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  12% (6 pods) t3.medium/$0.0520 On-Demand
ip-192-168-2-185.ap-northeast-2.compute.internal cpu ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  12% (7 pods) t3.medium/$0.0520 On-Demand
Press any key to quit

명령 샘플
# Standard usage
./eks-node-viewer

# Display both CPU and Memory Usage
./eks-node-viewer --resources cpu,memory

# Karenter nodes only
./eks-node-viewer --node-selector "karpenter.sh/provisioner-name"

# Display extra labels, i.e. AZ
./eks-node-viewer --extra-labels topology.kubernetes.io/zone

# Specify a particular AWS profile and region
AWS_PROFILE=myprofile AWS_REGION=us-west-2

기본 옵션
# select only Karpenter managed nodes
node-selector=karpenter.sh/provisioner-name

# display both CPU and memory
resources=cpu,memory
```

---

## HPA

### php-apache examples

* HPA 는 php-apache 를 실행 한 후에 HPA 객체를 생성, 그리고 부하를 주면 pod 가 증가 하는지를 확인한다. 

```sh
# Run and expose php-apache server
curl -s -O https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/php-apache.yaml
cat php-apache.yaml | yh
kubectl apply -f php-apache.yaml

# 확인
kubectl exec -it deploy/php-apache -- cat /var/www/html/index.php
...

# 모니터링 : 터미널2개 사용
watch -d 'kubectl get hpa,pod;echo;kubectl top pod;echo;kubectl top node'
kubectl exec -it deploy/php-apache -- top

# 접속
PODIP=$(kubectl get pod -l run=php-apache -o jsonpath={.items[0].status.podIP})
curl -s $PODIP; echo


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe hpa
Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler
Name:                                                  php-apache
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sat, 03 Jun 2023 12:40:03 +0900
Reference:                                             Deployment/php-apache
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       1 current / 1 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
Events:           <none>


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get hpa php-apache -o yaml | kubectl neat | yh
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache


    (yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# while true;do curl -s $PODIP; sleep 0.5; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!^C

# 아래 터미널에서 신규로 pod 가 생성됨을 확인 할 수 있다. 
Every 2.0s: kubectl get hpa,pod;echo;kubectl top pod;echo;kubectl top node                                     Sat Jun  3 12:42:48 2023

NAME                                             REFERENCE        	 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/php-apache   Deployment/php-apache   66%/50%   1         10        2          2m46s

NAME                              READY   STATUS    RESTARTS   AGE
pod/php-apache-698db99f59-6d6xf   1/1     Running   0          16s
pod/php-apache-698db99f59-dzp4s   1/1     Running   0          8m57s

NAMESPACE     NAME                                                        READY   STATUS    RESTARTS   AGE
default       php-apache-698db99f59-6d6xf                                 1/1     Running   0          28s
default       php-apache-698db99f59-dzp4s                                 1/1     Running   0          9m9s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

```

---

## KEDA

* Kubernetes Based Event Driven Autoscaler 이라고 하며
* 특정 이벤트를 기반으로 스케일 여부를 결정 할 수 있는 오픈소스 도구 이다. 
* overview diagram 을 보면 내부적으로는 HPA 기반으로 사용 하는것 같다. 



![keda-overview](/posts/study/cloudnet-aews/5w/images/keda-overview.png)


### KEDA install 

```sh 
cat <<EOT > keda-values.yaml
metricsServer:
  useHostNetwork: true

prometheus:
  metricServer:
    enabled: true
    port: 9022
    portName: metrics
    path: /metrics
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus Operator
      enabled: true
    podMonitor:
      # Enables PodMonitor creation for the Prometheus Operator
      enabled: true
  operator:
    enabled: true
    port: 8080
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus Operator
      enabled: true
    podMonitor:
      # Enables PodMonitor creation for the Prometheus Operator
      enabled: true

  webhooks:
    enabled: true
    port: 8080
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus webhooks
      enabled: true
EOT

kubectl create namespace keda
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --version 2.10.2 --namespace keda -f keda-values.yaml
```

### 설치 확인 

```sh 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get-all -n keda
NAME                                                                  NAMESPACE  AGE
configmap/kube-root-ca.crt                                            keda       65s
endpoints/keda-admission-webhooks                                     keda       60s
endpoints/keda-operator                                               keda       60s
endpoints/keda-operator-metrics-apiserver                             keda       60s
pod/keda-admission-webhooks-68cf687cbf-ld4tp                          keda       60s
pod/keda-operator-656478d687-59jdh                                    keda       60s
pod/keda-operator-metrics-apiserver-7fd585f657-2ch45                  keda       60s
secret/kedaorg-certs                                                  keda       49s
secret/sh.helm.release.v1.keda.v1                                     keda       61s
serviceaccount/default                                                keda       65s
serviceaccount/keda-operator                                          keda       61s
service/keda-admission-webhooks                                       keda       60s
service/keda-operator                                                 keda       60s
service/keda-operator-metrics-apiserver                               keda       60s
deployment.apps/keda-admission-webhooks                               keda       60s
deployment.apps/keda-operator                                         keda       60s
deployment.apps/keda-operator-metrics-apiserver                       keda       60s
replicaset.apps/keda-admission-webhooks-68cf687cbf                    keda       60s
replicaset.apps/keda-operator-656478d687                              keda       60s
replicaset.apps/keda-operator-metrics-apiserver-7fd585f657            keda       60s
lease.coordination.k8s.io/operator.keda.sh                            keda       49s
endpointslice.discovery.k8s.io/keda-admission-webhooks-rm6qh          keda       60s
endpointslice.discovery.k8s.io/keda-operator-gr2x2                    keda       60s
endpointslice.discovery.k8s.io/keda-operator-metrics-apiserver-wpds9  keda       60s
podmonitor.monitoring.coreos.com/keda-operator                        keda       60s
podmonitor.monitoring.coreos.com/keda-operator-metrics-apiserver      keda       60s
servicemonitor.monitoring.coreos.com/keda-admission-webhooks          keda       60s
servicemonitor.monitoring.coreos.com/keda-operator                    keda       60s
servicemonitor.monitoring.coreos.com/keda-operator-metrics-apiserver  keda       60s
rolebinding.rbac.authorization.k8s.io/keda-operator                   keda       60s
role.rbac.authorization.k8s.io/keda-operator                          keda       60s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get all -n keda
NAME                                                   READY   STATUS    RESTARTS      AGE
pod/keda-admission-webhooks-68cf687cbf-ld4tp           1/1     Running   0             61s
pod/keda-operator-656478d687-59jdh                     1/1     Running   1 (49s ago)   61s
pod/keda-operator-metrics-apiserver-7fd585f657-2ch45   1/1     Running   0             61s

NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
service/keda-admission-webhooks           ClusterIP   10.100.15.215    <none>        443/TCP,8080/TCP          61s
service/keda-operator                     ClusterIP   10.100.163.119   <none>        9666/TCP,8080/TCP         61s
service/keda-operator-metrics-apiserver   ClusterIP   10.100.114.247   <none>        443/TCP,80/TCP,9022/TCP   61s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-admission-webhooks           1/1     1            1           61s
deployment.apps/keda-operator                     1/1     1            1           61s
deployment.apps/keda-operator-metrics-apiserver   1/1     1            1           61s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-admission-webhooks-68cf687cbf           1         1         1       61s
replicaset.apps/keda-operator-656478d687                     1         1         1       61s
replicaset.apps/keda-operator-metrics-apiserver-7fd585f657   1         1         1       61s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get validatingwebhookconfigurations keda-admission
NAME             WEBHOOKS   AGE
keda-admission   1          63s
```

# KEDA Cron base Scale 

```sh
kubectl apply -f php-apache.yaml -n keda
kubectl get pod -n keda

cat <<EOT > keda-cron.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: php-apache-cron-scaled
spec:
  minReplicaCount: 0
  maxReplicaCount: 2
  pollingInterval: 30
  cooldownPeriod: 300
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Seoul
      start: 00,15,30,45 * * * *
      end: 05,20,35,50 * * * *
      desiredReplicas: "1"
EOT
kubectl apply -f keda-cron.yaml -n keda

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get scaledobject,hpa,pod -n keda
NAME                                          SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   TRIGGERS   AUTHENTICATION   READY   ACTIVE   FALLBACK   AGE
scaledobject.keda.sh/php-apache-cron-scaled   apps/v1.Deployment   php-apache        0     2     cron                        True    True     Unknown    4m23s

NAME                                                                  REFERENCE               TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/keda-hpa-php-apache-cron-scaled   Deployment/php-apache   <unknown>/1 (avg)   1         2         1          4m22s

NAME                                                   READY   STATUS    RESTARTS        AGE
pod/keda-admission-webhooks-68cf687cbf-ld4tp           1/1     Running   0               6m59s
pod/keda-operator-656478d687-59jdh                     1/1     Running   1 (6m47s ago)   6m59s
pod/keda-operator-metrics-apiserver-7fd585f657-2ch45   1/1     Running   0               6m59s

# 시간이 지남에 따라 php 가 생성된다. 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get po -n keda
NAME                                               READY   STATUS    RESTARTS       AGE
php-apache-698db99f59-x7zll                        1/1     Running   0              44s

```

### KEDA 삭제 

```sh
kubectl delete -f keda-cron.yaml -n keda && kubectl delete deploy php-apache -n keda && helm uninstall keda -n keda
kubectl delete namespace keda
```

---

## Vertical Pod Autoscaler 

* pod resporces.request 를 최적값으로 수정해준다. 
* HPA 와 병행 사용 불가능하며 VPA 에서 request 를 수정시에 파드를 재시작 한다

### VPA 설치 

```sh

# 코드 다운로드
git clone https://github.com/kubernetes/autoscaler.git
cd ~/autoscaler/vertical-pod-autoscaler/
tree hack

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# tree hack
hack
├── boilerplate.go.txt
├── convert-alpha-objects.sh
├── deploy-for-e2e.sh
├── generate-crd-yaml.sh
├── run-e2e.sh
├── run-e2e-tests.sh
├── update-codegen.sh
├── update-kubernetes-deps-in-e2e.sh
├── update-kubernetes-deps.sh
├── verify-codegen.sh
├── vpa-apply-upgrade.sh
├── vpa-down.sh
├── vpa-process-yaml.sh
├── vpa-process-yamls.sh
├── vpa-up.sh
└── warn-obsolete-vpa-objects.sh

0 directories, 16 files


# openssl 버전 확인
openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017

# openssl 1.1.1 이상 버전 확인
yum install openssl11 -y
openssl11 version
OpenSSL 1.1.1g FIPS  21 Apr 2020

# 스크립트파일내에 openssl11 수정
sed -i 's/openssl/openssl11/g' ~/autoscaler/vertical-pod-autoscaler/pkg/admission-controller/gencerts.sh

# Deploy the Vertical Pod Autoscaler to your cluster with the following command.
watch -d kubectl get pod -n kube-system
cat hack/vpa-up.sh

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# cat hack/vpa-up.sh
#!/bin/bash

# Copyright 2018 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname ${BASH_SOURCE})/..

$SCRIPT_ROOT/hack/vpa-process-yamls.sh create $*

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# ./hack/vpa-up.sh
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalercheckpoints.autoscaling.k8s.io created
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalers.autoscaling.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:vpa-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:evictioner created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-actor created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-target-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-target-reader-binding created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-evictioner-binding created
serviceaccount/vpa-admission-controller created
serviceaccount/vpa-recommender created
serviceaccount/vpa-updater created
clusterrole.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrole.rbac.authorization.k8s.io/system:vpa-status-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-status-reader-binding created
deployment.apps/vpa-updater created
deployment.apps/vpa-recommender created
Generating certs for the VPA Admission Controller in /tmp/vpa-certs.
Generating RSA private key, 2048 bit long modulus (2 primes)
..........+++++
..................+++++
e is 65537 (0x010001)
Can't load /root/.rnd into RNG
140628674213696:error:2406F079:random number generator:RAND_load_file:Cannot open file:crypto/rand/randfile.c:98:Filename=/root/.rnd
Generating RSA private key, 2048 bit long modulus (2 primes)
..............................................+++++
....................+++++
e is 65537 (0x010001)
Signature ok
subject=CN = vpa-webhook.kube-system.svc
Getting CA Private Key
Uploading certs to the cluster.
secret/vpa-tls-certs created
Deleting /tmp/vpa-certs.
deployment.apps/vpa-admission-controller created
service/vpa-webhook created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl get crd | grep autoscaling
verticalpodautoscalercheckpoints.autoscaling.k8s.io   2023-06-03T04:52:34Z
verticalpodautoscalers.autoscaling.k8s.io             2023-06-03T04:52:34Z
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl get mutatingwebhookconfigurations vpa-webhook-config
NAME                 WEBHOOKS   AGE
vpa-webhook-config   1          4s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl get mutatingwebhookconfigurations vpa-webhook-config -o json | jq
{
  "apiVersion": "admissionregistration.k8s.io/v1",
  "kind": "MutatingWebhookConfiguration",
  "metadata": {
    "creationTimestamp": "2023-06-03T04:53:00Z",
    "generation": 1,
    "name": "vpa-webhook-config",
    "resourceVersion": "25616",
    "uid": "eee41c51-b6d8-4a5d-ad43-370bcd88c512"
  },
  "webhooks": [
    {
      "admissionReviewVersions": [
        "v1"
      ],
      "clientConfig": {
        "caBundle": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURMVENDQWhXZ0F3SUJBZ0lVRGhVb3lYU0pIY3pCQnFDcnlIOEphZEZYZy9nd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0dURVhNQlVHQTFVRUF3d09kbkJoWDNkbFltaHZiMnRmWTJFd0lCY05Nak13TmpBek1EUTFNalF5V2hnUApNakk1TnpBek1UZ3dORFV5TkRKYU1Ca3hGekFWQmdOVkJBTU1Eblp3WVY5M1pXSm9iMjlyWDJOaE1JSUJJakFOCkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXJDa2RiK3B5akswWGRtS1VuUnQ3amZDNFpURlIKK1E5V0MzZU5mdXNWZkQ1Mm1scmlHMUpFSlhiMCtvUVpzTTdPWXZPL1VReGtnUlFLQnp3TXp1ZUtTanQ0L3dQMApwK1R6YzRZQ1hDZndqbU81cy9EalBHeCs2RHpwRHJDZ3hwbTZDVGw5OUVOOWxmdnFqbHNaZUU1Ly9ZZEZiRWRECnB4TnFEZHEvVU1rN25jTnl3WDYwTDA5ZjVmaHVtMEJ6bDk3VkJLZGp6cWtvaUdEeXVvUzZNYTFJdVlJZ294anEKQURXRzN1clkzS1hNM0VDOE90cFkvbEtlYzJ1MnUraFhneVBKTUtiMHU3cUsvMWRQWmh4dm15Nmk2RFNuWHBXcgpNNG96cWN3MXBTR0krNW90MS8rdUtrTitGSTJlZlNUUjJZMUs2dlZwS1A1SHBoSm0weSt1eFRMZzBRSURBUUFCCm8yc3dhVEFkQmdOVkhRNEVGZ1FVaWtHVTRaZ1VmSTR2WENydm12K3JMSE9UakRvd0h3WURWUjBqQkJnd0ZvQVUKaWtHVTRaZ1VmSTR2WENydm12K3JMSE9UakRvd0RBWURWUjBUQkFVd0F3RUIvekFaQmdOVkhSRUVFakFRZ2c1MgpjR0ZmZDJWaWFHOXZhMTlqWVRBTkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQWxKTHU2cG1TeW5kb0d1TkxNbHZICnZxUnVLTmRQakNqWnBxUmRYT0cwQjB1N2FlZ1h1MGhFN1JEVUN3NWsvQlg3OHZKeThtWEFLV0hPVnZ3dlBSMk8KQ216U1JjdUpodUZXUExMMWplWHNVYUlmVXN6SVRENndzOGZTR0s0TWMvVnZFQkI1S3ZKUFhHa1lPYVBrZWRlVwpxNlhtUWg4eHNid1lnRlk2eWdBMU1DOENUbUJYakZ4NXFrT29pdDBDbFg0V252TC9vZjY4NS91L0swaVYxSUt3CnB4V1Iwak1yNVB0LzROSjFMNzhNRjZLa2RCdld2TFZTNnZiSGkxRCtZclo2OEU2S2tDWGxKeXppNGUwcEhVK24KZlB5RkE2SEcvaFFDUlFTRE5uWlRFUWV3VkxtY2grNXY4NHp0cGpvSHBNMlN1VlM2U05hQUJFSmpsOTZWMHo2bQowUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K",
        "service": {
          "name": "vpa-webhook",
          "namespace": "kube-system",
          "port": 443
        }
      },
      "failurePolicy": "Ignore",
      "matchPolicy": "Equivalent",
      "name": "vpa.k8s.io",
      "namespaceSelector": {},
      "objectSelector": {},
      "reinvocationPolicy": "Never",
      "rules": [
        {
          "apiGroups": [
            ""
          ],
          "apiVersions": [
            "v1"
          ],
          "operations": [
            "CREATE"
          ],
          "resources": [
            "pods"
          ],
          "scope": "*"
        },
        {
          "apiGroups": [
            "autoscaling.k8s.io"
          ],
          "apiVersions": [
            "*"
          ],
          "operations": [
            "CREATE",
            "UPDATE"
          ],
          "resources": [
            "verticalpodautoscalers"
          ],
          "scope": "*"
        }
      ],
      "sideEffects": "None",
      "timeoutSeconds": 30
    }
  ]
}

```

### 공식 예제로 조절 확인 

```sh
# 모니터링
watch -d kubectl top pod

# 공식 예제 배포
cd ~/autoscaler/vertical-pod-autoscaler/
cat examples/hamster.yaml | yh
kubectl apply -f examples/hamster.yaml && kubectl get vpa -w

# 파드 리소스 Requestes 확인
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl describe pod | grep Requests: -A2
    Requests:
      cpu:        627m      # 새로 생성되는 애
      memory:     262144k
--
    Requests:
      cpu:        100m      # 기존에 있던 애 
      memory:     50Mi
--
    Requests:
      cpu:        100m
      memory:     50Mi


# VPA에 의해 기존 파드 삭제되고 신규 파드가 생성됨
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get events --sort-by=".metadata.creationTimestamp" | grep VPA
66s         Normal   EvictedByVPA        pod/hamster-5bccbb88c6-l8jh7       Pod was evicted by VPA Updater to apply resource recommendation.
6s          Normal   EvictedByVPA        pod/hamster-5bccbb88c6-n58sr       Pod was evicted by VPA Updater to apply resource recommendation.

```

### 삭제 

```sh
kubectl delete -f examples/hamster.yaml && cd ~/autoscaler/vertical-pod-autoscaler/ && ./hack/vpa-down.sh
```

---

## KRR 

* KRR 은 Price 가 있는 사용제품으로 보이는데 링크만 남기고 skip 하겠다. 
* [KRR Getting Started](https://github.com/robusta-dev/krr#getting-started)
* [KRR Youtube](https://www.youtube.com/watch?v=uITOzpf82RY)

---

## Inplace Resource Size 

* [K8s Blog](https://kubernetes.io/blog/2023/05/12/in-place-pod-resize-alpha/)

---

## CA 

* Cluster Autoscale 를 배포 하면 EKS 클러스터 내에 cluster-autoscaler 파드가 배포 되며 
* CA 는 penging 상태인 파드가 존재 할 경우에 worker node 를 스케일 아웃 한다. 
* 특정 시간을 간격으로 사용률 확인하여 스케줄 인/아웃을 수행하며 
* AWS 의 ASG 를 이용하여 스케일을 적용한다고 한다. 


### CA 설정 및 적용

```sh 
# 이미 onclick 배포 스크립트에 포함이 되어있다함
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# aws ec2 describe-instances  --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Reservations[*].Instances[*].Tags[*]" --output yaml | yh
- - - Key: kubernetes.io/cluster/yjkim-eks
      Value: owned
    - Key: k8s.io/cluster-autoscaler/enabled
      Value: 'true'

# CA 는 ASG 의 deployment 의 다른 옵션을 선택 할 수 있다고 함
## One ASG
## Multiple ASG
## Auto-discovery
## Control-plane Node Setup

# ASG 정보 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# aws autoscaling describe-auto-scaling-groups     --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]"     --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7cc43f9a-32a5-7587-0045-09ee5b7893c1  |  3 |  3 |  3 |
+------------------------------------------------+----+----+----+

# MAX 수정 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].AutoScalingGroupName" --output text)
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 3 --desired-capacity 3 --max-size 6

# 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# aws autoscaling describe-auto-scaling-groups     --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]"     --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7cc43f9a-32a5-7587-0045-09ee5b7893c1  |  3 |  6 |  3 |
+------------------------------------------------+----+----+----+

# CA 배포 
curl -s -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
sed -i "s/<YOUR CLUSTER NAME>/$CLUSTER_NAME/g" cluster-autoscaler-autodiscover.yaml
kubectl apply -f cluster-autoscaler-autodiscover.yaml

# 배포 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl get pod -n kube-system | grep cluster-autoscaler
cluster-autoscaler-58595668d8-25tm2            1/1     Running   0          39s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl describe deployments.apps -n kube-system cluster-autoscaler
Name:                   cluster-autoscaler
Namespace:              kube-system
CreationTimestamp:      Sat, 03 Jun 2023 14:13:02 +0900
Labels:                 app=cluster-autoscaler
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=cluster-autoscaler
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:           app=cluster-autoscaler
  Annotations:      prometheus.io/port: 8085
                    prometheus.io/scrape: true
  Service Account:  cluster-autoscaler
  Containers:
   cluster-autoscaler:
    Image:      registry.k8s.io/autoscaling/cluster-autoscaler:v1.26.2
    Port:       <none>
    Host Port:  <none>
    Command:
      ./cluster-autoscaler
      --v=4
      --stderrthreshold=info
      --cloud-provider=aws
      --skip-nodes-with-local-storage=false
      --expander=least-waste
      --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/yjkim-eks
    Limits:
      cpu:     100m
      memory:  600Mi
    Requests:
      cpu:        100m
      memory:     600Mi
    Environment:  <none>
    Mounts:
      /etc/ssl/certs/ca-certificates.crt from ssl-certs (ro)
  Volumes:
   ssl-certs:
    Type:               HostPath (bare host directory volume)
    Path:               /etc/ssl/certs/ca-bundle.crt
    HostPathType:
  Priority Class Name:  system-cluster-critical
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   cluster-autoscaler-58595668d8 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  40s   deployment-controller  Scaled up replica set cluster-autoscaler-58595668d8 to 1

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl describe deployments.apps -n kube-system cluster-autoscaler | grep node-group-auto-discovery
      --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/yjkim-eks

# (옵션) cluster-autoscaler 파드가 동작하는 워커 노드가 퇴출(evict) 되지 않게 설정
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

### Scale example 

```sh 

# 모니터링 
kubectl get nodes -w
while true; do kubectl get node; echo "------------------------------" ; date ; sleep 1; done
while true; do aws ec2 describe-instances --query "Reservations[*].Instances[*].{PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output text ; echo "------------------------------"; date; sleep 1; done

# Deploy a Sample App
# We will deploy an sample nginx application as a ReplicaSet of 1 Pod
cat <<EoF> nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF

kubectl apply -f nginx.yaml
kubectl get deployment/nginx-to-scaleout

# Scale our ReplicaSet
# Let’s scale out the replicaset to 15
kubectl scale --replicas=15 deployment/nginx-to-scaleout && date

# 확인
kubectl get pods -l app=nginx -o wide --watch
kubectl -n kube-system logs -f deployment/cluster-autoscaler

# 노드 자동 증가 확인
kubectl get nodes
aws autoscaling describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table


# 기존 
NAME                                               STATUS   ROLES    AGE    VERSION
ip-192-168-1-207.ap-northeast-3.compute.internal   Ready    <none>   134m   v1.24.13-eks-0a21954
ip-192-168-2-109.ap-northeast-3.compute.internal   Ready    <none>   134m   v1.24.13-eks-0a21954
ip-192-168-3-21.ap-northeast-3.compute.internal    Ready    <none>   134m   v1.24.13-eks-0a21954

# scaleout 됨을 확인 
NAME                                               STATUS   ROLES    AGE    VERSION
ip-192-168-1-207.ap-northeast-3.compute.internal   Ready    <none>   135m   v1.24.13-eks-0a21954
ip-192-168-2-109.ap-northeast-3.compute.internal   Ready    <none>   135m   v1.24.13-eks-0a21954
ip-192-168-2-144.ap-northeast-3.compute.internal   Ready    <none>   23s    v1.24.13-eks-0a21954
ip-192-168-3-21.ap-northeast-3.compute.internal    Ready    <none>   135m   v1.24.13-eks-0a21954
ip-192-168-3-251.ap-northeast-3.compute.internal   Ready    <none>   24s    v1.24.13-eks-0a21954


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# aws autoscaling describe-auto-scaling-groups \
>     --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
>     --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7cc43f9a-32a5-7587-0045-09ee5b7893c1  |  3 |  6 |  5 |
+------------------------------------------------+----+----+----+

```


### 삭제 

```sh 
kubectl delete -f nginx.yaml

# size 수정 
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 3 --desired-capacity 3 --max-size 3
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table

# Cluster Autoscaler 삭제
kubectl delete -f cluster-autoscaler-autodiscover.yaml
```

### 참고 링크 

* Cluster Over-Provisioning [workshop](https://www.eksworkshop.com/docs/autoscaling/compute/cluster-autoscaler/overprovisioning/)

### 문제점 

* 하나의 자원에 관리 방법의 분할 : ASG, EKS 
  * 관리정보가 서로 동기화 되지 않아 다양한 문제 발생 
* ASG 에만 의존하고 노드 생성 삭제등에 직접 관여 안함
* EKS 에서 노드 삭제 해도 EC2 인스턴스는 삭제 안됨 
* 노드 축소시 특정 노드를 축소 하도록 하기 어려움 
  * 예시) pod 가 적은노드 먼저 축소, 혹은 중요도가 적은 노드 축소, 혹은 이미 드래인된 노드 먼저 축소 
* 특정 노드 삭제시 동시 노드 개수 줄이기 어려움 
  * 정책 미지원 시 삭제 방식(예시) : 100대 중 미삭제 EC2 보호 설정 후 삭제 될 ec2의 파드를 이주 후 scaling 조절로 삭제 후 원복
* 특정 노드를 삭제 하면서 동시에 노드개수 줄이기 어려움 
* 폴링 방식이라 확장 여유를 확인 하면서 API 제한에 도달할 수 있음
* 스케일링 속도가 기본적으로 느림 
* punding 상태의 파드가 생기는 타이밍에 동작한다. 
  * request,limit 이 적절하지 않다면 실제 노드 부하 평균이 낮은 상황에서 스케일 아웃 되거나 부하 평균이 높은 상황에도 스케일 아웃이 되지 않는 현상 
* 리소스에 의한 스케줄링이 requests 를 기반으로 이루어짐, 실제 사용량으로 고려되지 않는다. 
  * request 는 낮고 limit 이 높다면 스케줄링이 가능하기에 클러스터가 스케일아웃이 동작하지 않음

---

## CPA 

* cluster profortional Autoscaler 
* 노드수 증가에 비례 성능처리가 필요한 어플리케이션을 수평으로 자동 확장 : 예시 coredns 

```sh 

helm repo add cluster-proportional-autoscaler https://kubernetes-sigs.github.io/cluster-proportional-autoscaler

# CPA 대상이 될 nginx 디플로이먼트 배포
cat <<EOT > cpa-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            cpu: "100m"
            memory: "64Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        ports:
        - containerPort: 80
EOT
kubectl apply -f cpa-nginx.yaml

# CPA 규칙 설정
cat <<EOF > cpa-values.yaml
config:
  ladder:
    nodesToReplicas:
      - [1, 1]
      - [2, 2]
      - [3, 3]
      - [4, 3]
      - [5, 5]
options:
  namespace: default
  target: "deployment/nginx-deployment"
EOF

# 모니터링
watch -d kubectl get pod

# helm 업그레이드
helm upgrade --install cluster-proportional-autoscaler -f cpa-values.yaml cluster-proportional-autoscaler/cluster-proportional-autoscaler

# 노드 5개로 증가
export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].AutoScalingGroupName" --output text)
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 5 --desired-capacity 5 --max-size 5
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='yjkim-eks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table

# 노드가 증가함에 따라 nginx 도 증가됨을 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 vertical-pod-autoscaler]# kubectl get po -l app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-858477475d-2dhzx   1/1     Running   0          82s
nginx-deployment-858477475d-d4kps   1/1     Running   0          2m42s
nginx-deployment-858477475d-ftm86   1/1     Running   0          3m48s
nginx-deployment-858477475d-hmwvp   1/1     Running   0          2m42s
nginx-deployment-858477475d-v8pdh   1/1     Running   0          82s

# 삭제 
helm uninstall cluster-proportional-autoscaler && kubectl delete -f cpa-nginx.yaml
```

---

## karpenter 사전 정리 

```sh
helm uninstall -n kube-system kube-ops-view
helm uninstall -n monitoring kube-prometheus-stack
eksctl delete cluster --name $CLUSTER_NAME && aws cloudformation delete-stack --stack-name $CLUSTER_NAME
```

---

## Karpenter 

### 개요 

* 작동 방식 
  * Provisioning : 모니터링 -> 스케줄링 안된 포드 발견 -> 평가 -> 생성 
  * Deprovisioning : 모니터링 -> 비어있는 노드 발견 -> 제거 
* Provisioner CRD : 시작 템플릿의 설정을 대신함 
  * 필수 : 보안그룹, 서브넷
  * 리소스 찾는 방식 : 태그기반 자동, 리소스 직접 명시 
  * 인스턴스 타입은 가드레일 방식으로 선언 가능 : 스팟(우선) vs 온디멘드 
* Pod 에 적합한 인스턴스 중 가장 저렴한 인스턴스로 증설 
* 노드를 줄여도 다른 노드에 충분한 여유가 있다면 자동으로 정리해줌 
* 큰 노드 하나가 작은 노드 여러개보다 비용이 저렴하다면 자동으로 합쳐줌
* 오버 프로비저닝 필요 
  * 카펜터 쓰더라도 EC2 실행, 데몬셋 모두 설치 되는데 1~2분 정도 소요되어서 깡통 증설용 Pod 만들 필요 있음 
  * karpenter + KEDA 조합으로 오버프로비저닝 가능

* 위 글만 보면 기승전 karpenter 로 단결 되는 느낌이다.

### 실습환경 배포 

```sh 
# YAML 파일 다운로드
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/karpenter-preconfig.yaml

# Cloudformation skip 

# IP 주소 확인 : 172.30.0.0/16 VPC 대역에서 172.30.1.0/24 대역을 사용 중
ip -br -c addr
```

### EKS 배포 

```sh 
# 환경변수 정보 확인
export | egrep 'ACCOUNT|AWS_|CLUSTER' | egrep -v 'SECRET|KEY'

# 환경변수 설정
export KARPENTER_VERSION=v0.27.5
export TEMPOUT=$(mktemp)
echo $KARPENTER_VERSION $CLUSTER_NAME $AWS_DEFAULT_REGION $AWS_ACCOUNT_ID $TEMPOUT

# CloudFormation 스택으로 IAM Policy, Role, EC2 Instance Profile 생성 : 3분 정도 소요
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

# 클러스터 생성 : myeks2 EKS 클러스터 생성 19분 정도 소요
eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.24"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: ${CLUSTER_NAME}-karpenter
    attachPolicyARNs:
    - arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
    roleOnly: true

iamIdentityMappings:
- arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
- instanceType: m5.large
  amiFamily: AmazonLinux2
  name: ${CLUSTER_NAME}-ng
  desiredCapacity: 2
  minSize: 1
  maxSize: 10
  iam:
    withAddonPolicies:
      externalDNS: true

## Optionally run on fargate
# fargateProfiles:
# - name: karpenter
#  selectors:
#  - namespace: karpenter
EOF

# eks 배포 확인
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get cluster
NAME		REGION		EKSCTL CREATED
yjkim-eks	ap-northeast-3	True
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get nodegroup --cluster $CLUSTER_NAME
CLUSTER		NODEGROUP	STATUS	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID	ASG NAME						TYPE
yjkim-eks	yjkim-eks-ng	ACTIVE	2023-06-03T07:19:28Z	1		10		2			m5.large	AL2_x86_64	eks-yjkim-eks-ng-3ec44010-06b2-4adc-1e0e-940485e4b394	managed
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get iamidentitymapping --cluster $CLUSTER_NAME
ARN												USERNAME				GROUPS					ACCOUNT
arn:aws:iam::123123123:role/KarpenterNodeRole-yjkim-eks					system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes
arn:aws:iam::123123123:role/eksctl-yjkim-eks-nodegroup-yjkim-NodeInstanceRole-17PFEQAAJPFM4	system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get iamserviceaccount --cluster $CLUSTER_NAME
NAMESPACE	NAME		ROLE ARN
karpenter	karpenter	arn:aws:iam::123123123:role/yjkim-eks-karpenter
kube-system	aws-node	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-5PHPGY2OZNTQ
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get addon --cluster $CLUSTER_NAME
2023-06-03 16:23:09 [ℹ]  Kubernetes version "1.24" in use by cluster "yjkim-eks"
2023-06-03 16:23:09 [ℹ]  getting all addons
No addons found

# K8s 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
NAME                                                STATUS   ROLES    AGE    VERSION                INSTANCE-TYPE   CAPACITYTYPE   ZONE
ip-192-168-27-61.ap-northeast-3.compute.internal    Ready    <none>   3m9s   v1.24.13-eks-0a21954   m5.large        ON_DEMAND      ap-northeast-3a
ip-192-168-87-247.ap-northeast-3.compute.internal   Ready    <none>   3m9s   v1.24.13-eks-0a21954   m5.large        ON_DEMAND      ap-northeast-3c
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -owide
NAME                       READY   STATUS    RESTARTS   AGE     IP               NODE                                                NOMINATED NODE   READINESS GATES
aws-node-2bzdj             1/1     Running   0          3m10s   192.168.87.247   ip-192-168-87-247.ap-northeast-3.compute.internal   <none>           <none>
aws-node-m8d4s             1/1     Running   0          3m10s   192.168.27.61    ip-192-168-27-61.ap-northeast-3.compute.internal    <none>           <none>
coredns-5bc54f57fc-8jkcc   1/1     Running   0          9m52s   192.168.12.25    ip-192-168-27-61.ap-northeast-3.compute.internal    <none>           <none>
coredns-5bc54f57fc-l9pbb   1/1     Running   0          9m52s   192.168.9.88     ip-192-168-27-61.ap-northeast-3.compute.internal    <none>           <none>
kube-proxy-927kg           1/1     Running   0          3m10s   192.168.27.61    ip-192-168-27-61.ap-northeast-3.compute.internal    <none>           <none>
kube-proxy-hkv59           1/1     Running   0          3m10s   192.168.87.247   ip-192-168-87-247.ap-northeast-3.compute.internal   <none>           <none>

```

### Karpenter 배포 

```sh
# Karpenter 변수 설정 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

# EC2 Spot Fleet 사용을 위한 service-linked-role 생성 확인 : 만들어있는것을 확인하는 거라 아래 에러 출력이 정상!
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

# docker logout : Logout of docker to perform an unauthenticated pull against the public ECR
docker logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get-all -n karpenter
NAME                                                 NAMESPACE  AGE
configmap/config-logging                             karpenter  52s
configmap/karpenter-global-settings                  karpenter  52s
configmap/kube-root-ca.crt                           karpenter  52s
endpoints/karpenter                                  karpenter  52s
pod/karpenter-657fc5fb7d-mrmsf                       karpenter  52s
pod/karpenter-657fc5fb7d-rxzgs                       karpenter  52s
secret/karpenter-cert                                karpenter  52s
secret/sh.helm.release.v1.karpenter.v1               karpenter  52s
serviceaccount/default                               karpenter  52s
serviceaccount/karpenter                             karpenter  52s
service/karpenter                                    karpenter  52s
deployment.apps/karpenter                            karpenter  52s
replicaset.apps/karpenter-657fc5fb7d                 karpenter  52s
lease.coordination.k8s.io/karpenter-leader-election  karpenter  44s
endpointslice.discovery.k8s.io/karpenter-46szw       karpenter  52s
poddisruptionbudget.policy/karpenter                 karpenter  52s
rolebinding.rbac.authorization.k8s.io/karpenter      karpenter  52s
role.rbac.authorization.k8s.io/karpenter             karpenter  52s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get all -n karpenter
NAME                             READY   STATUS    RESTARTS   AGE
pod/karpenter-657fc5fb7d-mrmsf   1/1     Running   0          53s
pod/karpenter-657fc5fb7d-rxzgs   1/1     Running   0          53s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)            AGE
service/karpenter   ClusterIP   10.100.62.222   <none>        8080/TCP,443/TCP   53s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/karpenter   2/2     2            2           53s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/karpenter-657fc5fb7d   2         2         2       53s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get cm -n karpenter karpenter-global-settings -o jsonpath={.data} | jq
{
  "aws.clusterEndpoint": "",
  "aws.clusterName": "yjkim-eks",
  "aws.defaultInstanceProfile": "KarpenterNodeInstanceProfile-yjkim-eks",
  "aws.enableENILimitedPodDensity": "true",
  "aws.enablePodENI": "false",
  "aws.interruptionQueueName": "yjkim-eks",
  "aws.isolatedVPC": "false",
  "aws.nodeNameConvention": "ip-name",
  "aws.vmMemoryOverheadPercent": "0.075",
  "batchIdleDuration": "1s",
  "batchMaxDuration": "10s",
  "featureGates.driftEnabled": "false"
}
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get crd | grep karpenter
awsnodetemplates.karpenter.k8s.aws           2023-06-03T07:27:38Z
provisioners.karpenter.sh                    2023-06-03T07:27:38Z

```

### Provisioner 실습 

* create Provisioner

```sh
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF

# 확인
kubectl get awsnodetemplates,provisioners

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get awsnodetemplates,provisioners
NAME                                        AGE
awsnodetemplate.karpenter.k8s.aws/default   2s

NAME                               AGE
provisioner.karpenter.sh/default   2s


# pause 파드 1개에 CPU 1개 최소 보장 할당
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

aws ec2 describe-spot-instance-requests --filters "Name=state,Values=active" --output table
kubectl get node -l karpenter.sh/capacity-type=spot -o jsonpath='{.items[0].metadata.labels}' | jq
kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type,node.kubernetes.io/instance-type
NAME                                                STATUS   ROLES    AGE   VERSION                CAPACITYTYPE   CAPACITY-TYPE   I
NSTANCE-TYPE
ip-192-168-121-65.ap-northeast-3.compute.internal   Ready    <none>   61s   v1.24.13-eks-0a21954                  spot  	  c
5n.2xlarge
ip-192-168-27-61.ap-northeast-3.compute.internal    Ready    <none>   15m   v1.24.13-eks-0a21954   ON_DEMAND                      m
5.large
ip-192-168-87-247.ap-northeast-3.compute.internal   Ready    <none>   15m   v1.24.13-eks-0a21954   ON_DEMAND                      m
5.large
```


### Consolidation 

```sh 
kubectl delete provisioners default
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  consolidation:
    enabled: true
  labels:
    type: karpenter
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  providerRef:
    name: default
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values:
        - on-demand
    - key: node.kubernetes.io/instance-type
      operator: In
      values:
        - c5.large
        - m5.large
        - m5.xlarge
EOF

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 12
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

# 인스턴스 확인
# This changes the total memory request for this deployment to around 12Gi, 
# which when adjusted to account for the roughly 600Mi reserved for the kubelet on each node means that this will fit on 2 instances of type m5.large:
kubectl get node -l type=karpenter
kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type
kubectl get node --label-columns=node.kubernetes.io/instance-type,topology.kubernetes.io/zone

NAME                                                STATUS     ROLES    AGE   VERSION                CAPACITYTYPE   CAPACITY-TYPE   INSTANCE-TYPE
ip-192-168-18-45.ap-northeast-3.compute.internal    NotReady   <none>   33s   v1.24.13-eks-0a21954                  on-demand       m5.xlarge
ip-192-168-27-61.ap-northeast-3.compute.internal    Ready      <none>   19m   v1.24.13-eks-0a21954   ON_DEMAND                      m5.large
ip-192-168-53-104.ap-northeast-3.compute.internal   NotReady   <none>   33s   v1.24.13-eks-0a21954                  on-demand       m5.xlarge
ip-192-168-55-148.ap-northeast-3.compute.internal   NotReady   <none>   33s   v1.24.13-eks-0a21954                  on-demand       m5.xlarge
ip-192-168-7-89.ap-northeast-3.compute.internal     NotReady   <none>   33s   v1.24.13-eks-0a21954                  on-demand       m5.xlarge
ip-192-168-87-247.ap-northeast-3.compute.internal   Ready      <none>   19m   v1.24.13-eks-0a21954   ON_DEMAND                      m5.large


# Next, scale the number of replicas back down to 5:
kubectl scale deployment inflate --replicas 5

# The output will show Karpenter identifying specific nodes to cordon, drain and then terminate:
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-06-03T07:41:08.087Z	INFO	controller.termination	cordoned node	{"commit": "698f22f-dirty", "node": "ip-192-168-53-104.ap-northeast-3.compute.internal"}
2023-06-03T07:41:08.356Z	INFO	controller.termination	deleted node	{"commit": "698f22f-dirty", "node": "ip-192-168-53-104.ap-northeast-3.compute.internal"}

# Next, scale the number of replicas back down to 1
kubectl scale deployment inflate --replicas 1
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-06-03T07:41:52.201Z	INFO	controller.deprovisioning	deprovisioning via consolidation delete, terminating 1 machines ip-192-168-55-148.ap-northeast-3.compute.internal/m5.xlarge/on-demand	{"commit": "698f22f-dirty"}
2023-06-03T07:41:52.232Z	INFO	controller.termination	cordoned node	{"commit": "698f22f-dirty", "node": "ip-192-168-55-148.ap-northeast-3.compute.internal"}
2023-06-03T07:41:52.501Z	INFO	controller.termination	deleted node	{"commit": "698f22f-dirty", "node": "ip-192-168-55-148.ap-northeast-3.compute.internal"}

# 인스턴스 확인
kubectl get node -l type=karpenter
kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type
kubectl get node --label-columns=node.kubernetes.io/instance-type,topology.kubernetes.io/zone

NAME                                                STATUS                     ROLES    AGE     VERSION                CAPACITYTYPE   CAPACITY-TYPE   INSTANCE-TYPE
ip-192-168-189-50.ap-northeast-3.compute.internal   Unknown                    <none>   5s                                            on-demand       c5.large
ip-192-168-27-61.ap-northeast-3.compute.internal    Ready                      <none>   21m     v1.24.13-eks-0a21954   ON_DEMAND                      m5.large
ip-192-168-7-89.ap-northeast-3.compute.internal     Ready,SchedulingDisabled   <none>   2m14s   v1.24.13-eks-0a21954                  on-demand       m5.xlarge
ip-192-168-87-247.ap-northeast-3.compute.internal   Ready                      <none>   21m     v1.24.13-eks-0a21954   ON_DEMAND                      m5.large

# 삭제
kubectl delete deployment inflate

```

### 리소스 삭제 

```sh 
helm uninstall karpenter --namespace karpenter

# 위 삭제 완료 후 아래 삭제
aws ec2 describe-launch-templates --filters Name=tag:karpenter.k8s.aws/cluster,Values=${CLUSTER_NAME} |
    jq -r ".LaunchTemplates[].LaunchTemplateName" |
    xargs -I{} aws ec2 delete-launch-template --launch-template-name {}

# 클러스터 삭제
eksctl delete cluster --name "${CLUSTER_NAME}"

#
aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"

# 위 삭제 완료 후 아래 삭제
aws cloudformation delete-stack --stack-name ${CLUSTER_NAME}
```