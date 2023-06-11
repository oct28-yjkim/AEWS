---
title: "EKS 의 network 를 알아봅시다. "
date: 2023-05-07T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 2w
    identifier: 2w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 

2주차 세션 진행이 며 PKOS 의 내용과 많은 부분이 겹치는 주제가 많다. 

AWS 에서 K8S 를 이용하기 위해서는 당연히도 네트워크 리소스에 대한 사용이 필연 적이며 
굳이 VPC 를 구성하고 EC2 를 생성한 후에  그 위에 K8S 를 실행 하기 보다는 
K8S 자체에서 제공하는 API 들을 이용하여 AWS 에서 K8S 를 사용할 수 있는 SAAS 제품 들이 있고 그게 EKS 이다. 
EKS 에서 K8S 의 CNI 를 어떻게 구성하고 인프라를 정의 할 수 있는지. 
K8S 의 Service, Ingress 등을 어떻게 외부로 노출(Expose) 할 수 있을지? 
그리고 해당 객체들을 AWS 내부 인프라 들을 이용해서 어떻게 활용 할 수 있을지 에 대하여 이번 주차에 알아보기로 한다. 

이번 주 주제는 
* AWS-VPC-CNI 알아보기 
* EKS 상에서 K8S Service, Ingress 객체에 대해 알아보기 
* 도전과제 진행해보기 

위 내용 이지만 가정의 주간으로 시간을 많이 내지 못하여 도전과제 위주로 진행 하겠다. 

---

## EKS 배포 

우선 기존에 있던 원클릭 배포를 이용하여 배포. 

### EKS LoadBalancer Controller 배포 

```sh
# IAM Policy 생성 
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# 생성된 IAM Policy Arn 확인
aws iam list-policies --scope Local
aws iam get-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
aws iam get-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.Arn'

# Service Account 생성 
eksctl create iamserviceaccount --cluster=$CLUSTER_NAME --namespace=kube-system --name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve

# IRSA 정보 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME

# 서비스 어카운트 확인
kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml | yh

# Helm chart 로 설치 
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

# NLB 배포 확인 
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/2/echo-service-nlb.yaml
cat echo-service-nlb.yaml | yh
kubectl apply -f echo-service-nlb.yaml

# 확인
kubectl get deploy,pod
kubectl get svc,ep,ingressclassparams,targetgroupbindings
kubectl get targetgroupbindings -o json | jq

# AWS ELB(NLB) 정보 확인
aws elbv2 describe-load-balancers | jq
aws elbv2 describe-load-balancers --query 'LoadBalancers[*].State.Code' --output text

# 웹 접속 주소 확인
(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get svc svc-nlb-ip-type -o jsonpath={.status.loadBalancer.ingress[0].hostname} | awk '{ print "Pod Web URL = http://"$1 }'
Pod Web URL = http://k8s-default-svcnlbip-d5b4fd2691-45bd3b9de5c76410.elb.ap-northeast-2.amazonaws.com

```

### EKS External DNS 배포 


```sh

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# MyDomain=oct28-yjkim.ml

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." | jq
{
  "HostedZones": [
    {
      "Id": "/hostedzone/Z02035442Y4138L41YZSE",
      "Name": "oct28-yjkim.ml.",
      "CallerReference": "3fd527f3-639c-4fa7-a7a9-890db44fe98f",
      "Config": {
        "Comment": "",
        "PrivateZone": false
      },
      "ResourceRecordSetCount": 6
    }
  ],
  "DNSName": "oct28-yjkim.ml.",
  "IsTruncated": false,
  "MaxItems": "100"
}

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Name"
"oct28-yjkim.ml."

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text
/hostedzone/Z02035442Y4138L41YZSE

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# MyDnzHostedZoneId=`aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text`

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo $MyDnzHostedZoneId
/hostedzone/Z02035442Y4138L41YZSE

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat externaldns.yaml | yh
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: kube-system
  labels:
    app.kubernetes.io/name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods","nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: kube-system # change to desired namespace: externaldns, kube-addons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.13.4
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=${MyDomain} # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws

            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=${MyDnzHostedZoneId}
          env:
            - name: AWS_DEFAULT_REGION
              value: ap-northeast-2 # change to region where EKS is installed
(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -
serviceaccount/external-dns created
clusterrole.rbac.authorization.k8s.io/external-dns created
clusterrolebinding.rbac.authorization.k8s.io/external-dns-viewer created
deployment.apps/external-dns created

# 배포 검증용 테트리스 앱 배포 

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tetris
  labels:
    app: tetris
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tetris
  template:
    metadata:
      labels:
        app: tetris
    spec:
      containers:
      - name: tetris
        image: bsord/tetris
---
apiVersion: v1
kind: Service
metadata:
  name: tetris
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    #service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "80"
spec:
  selector:
    app: tetris
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  type: LoadBalancer
  loadBalancerClass: service.k8s.aws/nlb
EOF
deployment.apps/tetris created
service/tetris created

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl annotate service tetris "external-dns.alpha.kubernetes.io/hostname=tetris.$MyDomain"
service/tetris annotated

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo -e "My Domain Checker = https://www.whatsmydns.net/#A/tetris.$MyDomain"
My Domain Checker = https://www.whatsmydns.net/#A/tetris.oct28-yjkim.ml

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo -e "Tetris Game URL = http://tetris.$MyDomain"
Tetris Game URL = http://tetris.oct28-yjkim.ml
```

---

## 도전과제 

### DNS 조절 문제에 대한 메트릭 수집해보기 

* 참고 자료 링크 : https://aws.amazon.com/ko/blogs/mt/monitoring-coredns-for-dns-throttling-issues-using-aws-open-source-monitoring-services/

참고 자료에서 언급 했듯이 K8s 에서는 마이크로 서비스 환경에서 오케스트레이션 도구를 사용 했을 시에는 모니터링 해야 하는 부분이 많다고 언급 했으며 
특정 문제들에 대한 모니터링 및 Alarm 방법에 대하여 소개 하고 있다. 

CoreDNS 를 이용 할 시에는 DNS 의 문제들은 식벌하기 어려우며 CoreDNS 는 ENI 수준으로 설정된 
1024 PPS(초당 패킷 수) 의 엄격한 제한을 받고 있으며 엄격한 제한으로 인해 누락되는 트레픽이 발생 되는것을 감지하는 방법에 대한 이야기를 하고 있다. 

Worker 노드에서는 DNS 제한 문제를 식별 하기 위하여 ENA 드라이버를 이용하여 인스턴스의 네트워크 성능 메트릭을 게시 하고 있으며 
linklocal_allowance_exceeded 메트릭을 이용하여 DNS 쓰로틀링 문제를 확인 할 수 있다고 한다


![아키텍쳐](/posts/study/cloudnet-aews/2w/images/cloudops_1194_1.png)

우측에 표기된 Prometheus 는 PKOS 에서 작업한 Prometheus Stack 로 대체 할 예정이다. 

사전 조건으로 전달 받은 EKS Oneclick 배포를 통하여 EKS 를 진행 한다. 

### Exporter 배포 

아래 exporter 은 ethtools 라는 linux 네트워킹 유틸리티 의 값들을 Exporter 로 출력 해주는 도구이다. 
Manifest 를 배포 하게 된다면 각 노드마다 exporter 이 배포 되며 ENI 의 Metric 이 수집되는것을 확인 할 수 있다. 


```sh
kubectl create ns monitoring 

cat << EOF > ethtool-exporter.yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ethtool-exporter
  labels:
    app: ethtool-exporter
spec:
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 100%
  selector:
    matchLabels:
      app: ethtool-exporter
  template:
    metadata:
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9417'      
      labels:
        app: ethtool-exporter
    spec:
      hostNetwork: true
      terminationGracePeriodSeconds: 0
      containers:
      - name: ethtool-exporter
        env:
        - name: IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP      
        image: drdivano/ethtool-exporter@sha256:39e0916b16de07f62c2becb917c94cbb3a6e124a577e1325505e4d0cdd550d7b
        command:
          - "sh"
          - "-exc"
          - "python3 /ethtool-exporter.py -l \$(IP):9417 -I '(eth|em|eno|ens|enp)[0-9s]+'"
        ports:
        - containerPort: 9417
          hostPort: 9417
          name: http
          
        resources:
          limits:
            cpu: 250m
            memory: 100Mi
          requests:
            cpu: 10m
            memory: 50Mi

      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ethtool-exporter
  name: ethtool-exporter
spec:
  clusterIP: None
  ports:
    - name: http
      port: 9417
  selector:
    app: ethtool-exporter
EOF

kubectl apply -n monitoring -f ethtool-exporter.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
ethtool-exporter-jkfsz   1/1     Running   0          79m   192.168.1.154   ip-192-168-1-154.ap-northeast-3.compute.internal   <none>           <none>
ethtool-exporter-mjd2c   1/1     Running   0          79m   192.168.2.156   ip-192-168-2-156.ap-northeast-3.compute.internal   <none>           <none>
ethtool-exporter-z7dkp   1/1     Running   0          79m   192.168.3.227   ip-192-168-3-227.ap-northeast-3.compute.internal   <none>           <none>

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s 192.168.1.154:9417 | grep linklocal
node_net_ethtool{device="eth1",type="linklocal_allowance_exceeded"} 0.0
node_net_ethtool{device="eth0",type="linklocal_allowance_exceeded"} 0.0

```

### Prometheus 배포 

```sh
# install prometheus 
CERT_ARN=`aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn[]' --output text`
echo "alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN"

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

cat <<EOT > ~/monitor-values.yaml
grafana:
  defaultDashboardsTimezone: Asia/Seoul
  adminPassword: prom-operator

  ingress:
    enabled: true
    ingressClassName: alb

    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/group.name: "monitoring"

    hosts:
      - grafana.oct28-yjkim.ml

    paths:
      - /*

prometheus:
  ingress:
    enabled: true
    ingressClassName: alb

    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/group.name: "monitoring"

    hosts:
      - prometheus.oct28-yjkim.ml

    paths:
      - /*

  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 5d
    retentionSize: "10GiB"

alertmanager:
  enabled: false
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false
EOT

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 45.7.1 \
--set prometheus.prometheusSpec.scrapeInterval='15s' --set prometheus.prometheusSpec.evaluationInterval='15s' \
-f monitor-values.yaml --namespace monitoring

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm ls -n monitoring
NAME                 	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART                       	APP VERSION
kube-prometheus-stack	monitoring	1       	2023-05-09 01:32:24.597404583 +0900 KST	deployed	kube-prometheus-stack-45.7.1	v0.63.0

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get ing -A
NAMESPACE    NAME                               CLASS   HOSTS                       ADDRESS                                                                PORTS   AGE
monitoring   kube-prometheus-stack-grafana      alb     grafana.oct28-yjkim.ml      k8s-monitoring-589b3cf8d1-591538180.ap-northeast-2.elb.amazonaws.com   80      3m27s
monitoring   kube-prometheus-stack-prometheus   alb     prometheus.oct28-yjkim.ml   k8s-monitoring-589b3cf8d1-591538180.ap-northeast-2.elb.amazonaws.com   80      3m27s

```

### 파이프라인 구성 

기존에 Prometheus 의 ServiceMonitor 을 구성 해도 좋다. 
하지만 원문에서는 otel collector 이라는 도구를 이용하여 파이프라인이 구성 된다. 


![otel collector](/posts/study/cloudnet-aews/2w/images/otel-collector.svg)

* otel collector 
  * 기존에 opentraceing 과 opencensus 조직에서 병합 되어 opentelemetry 라는 조직으로 구성 된것으로 알고 있다. 
  * opentraceing 은 jaeger, zipkin 의 traceing 을 주로 담당하고 있으며 istio 에서도 사용이 가능하다. 
  * opencensus 는 google 에서 제작되어 opentelemetry 로 병합 된것으로만 알고 있다. 
  * otel collector 을 이용하여 metric, log, tracing 을 1개의 도구로 모니터링 할 수 있다. 
  * 하지만 거의 beta 수준으로 개발 되어있는 것으로 알고 있다. 
  * 계측기 라는 이름으로 contribute 된 repo 등이 있으며 해당 계측기에서 다양한 툴들을 지원 해준다. (예시 : nginx, 혹은 다른것들, 웹 어플리케이션 django ... )
  * istio 의 envoy 는 최근에는 opentracing 으로 알고 있다. 
  * 기존 opentracing 에서 opentracing 으로의 변경이 UDP, HTTP 프로토콜에서 gRPC 를 이용한 spec 를 지원 하고 있다. 
  * otel collector 의 구성은 service, receiver, processor, exporter 로 구성이 되어있고 
  * service 에서는 receiver, processor, exporter 를 명시 하도록 구성되어있으며 : 1 or N 개의 pipeline 정의 
  * receiver 은 metric 을 수집 혹은 다른 툴에서 전달 받는 ingest 영역이라고 보면 된다. 아래 예시에서는 exporter 에서 수집 하도록 구성되어있다. 
  * processor 는 receiver 에서 전달 받은 것을 exporter 를 통하여 전달 하는 양식을 명시 한다. : 배치작업이나 몇개의 건을 보낼것인가 등등을 정의 
  * exporter 는 다른 storage 혹은 cloud 벤더사로 전달 하여 모니터링 할 수 있는 양식을 명시 한다. : 예시에서는 prometheus 로 push 방식으로 전달한다. 
    * 프로메테우스에서는 remote write 를 통한 push 방식을 추천하지는 않으며 1개의 도구로 여러가지 데이터를 보낼수 있는 도구다 라고 알면 될듯하다. 



```sh
# install otel permission 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f https://amazon-eks.s3.amazonaws.com/docs/addons-otel-permissions.yaml
namespace/opentelemetry-operator-system created
clusterrole.rbac.authorization.k8s.io/eks:addon-manager-otel created
clusterrolebinding.rbac.authorization.k8s.io/eks:addon-manager-otel created
role.rbac.authorization.k8s.io/eks:addon-manager created
rolebinding.rbac.authorization.k8s.io/eks:addon-manager created

# install adot addon 
aws eks describe-cluster --name yjkim-eks
aws eks create-addon --addon-name adot --cluster-name yjkim-eks --configuration-values my-configuration-values

# install certmanager 
kubectl apply -f https://github.com/cert-manager/cert- manager/releases/download/v1.8.2/cert-manager.yaml
namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
configmap/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrole.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

# IAM 생성 
# AMP 에 전달 하는 용 이긴 한데 예시에서 사용 하고 있어서 생성 해준다. 
eksctl create iamserviceaccount \
  --name amp-iamproxy-ingest-role \
  --namespace monitoring \
  --region ap-northeast-2 \
  --cluster $EKS_CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
  --approve \
  --override-existing-serviceaccounts

# 링크에서 전달 받은 예시는 AMP 예시 이기에 변경이 필요 하다. 
# 제거 할 내용은 aws extension 을 제거하고 
# amp 가 아닌 prometheus 를 보도록 구성 하면 된다. 
export AMP_REMOTE_WRITE_ENDPOINT=http://kube-prometheus-stack-prometheus:9090/api/v1/writ
# AS IS : 제거 대상 
cat > collector-config-amp.yaml <<EOF
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: my-collector-amp
... 
    extensions:
      sigv4auth:
        region: $AWS_REGION
        service: "aps"
...
      prometheus:
        config:
          global:
            scrape_interval: 60s
            scrape_timeout: 30s
            external_labels:              # 있어도 그만 없어도 그만이다. 
              cluster: $EKS_CLUSTER_NAME
...
    exporters:
      prometheusremotewrite:
        endpoint: $AMP_REMOTE_WRITE_ENDPOINT
        auth:                             # 제거 
          authenticator: sigv4auth        # 제거 

    service:
      extensions: [sigv4auth] # 제거 
      pipelines:   
        metrics:
          receivers: [prometheus]
          processors: [batch/metrics]
          exporters: [prometheusremotewrite]
EOF


# TO BE 
cat > collector-config-amp.yaml <<EOF
#
# OpenTelemetry Collector configuration
# Replace the variables REGION and WORKSPACE per your target environment
# Metrics pipeline with Prometheus Receiver and AWS Remote Write Exporter sending metrics to Amazon Managed Prometheus
#
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: observability
  namespace: monitoring
spec:
  image: public.ecr.aws/aws-observability/aws-otel-collector:v0.20.0
  mode: deployment
  serviceAccount: amp-iamproxy-ingest-role

  config: |

    receivers:
      prometheus:
        config:
          global:
            scrape_interval: 15s
            scrape_timeout: 10s

          scrape_configs:
          - job_name: kubernetes-apiservers
            bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
            kubernetes_sd_configs:
            - role: endpoints
            relabel_configs:
            - action: keep
              regex: default;kubernetes;https
              source_labels:
              - __meta_kubernetes_namespace
              - __meta_kubernetes_service_name
              - __meta_kubernetes_endpoint_port_name
            scheme: https
            tls_config:
              ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
              insecure_skip_verify: true

          - job_name: kubernetes-nodes
            bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
            kubernetes_sd_configs:
            - role: node
            relabel_configs:
            - action: labelmap
              regex: __meta_kubernetes_node_label_(.+)
            - replacement: kubernetes.default.svc:443
              target_label: __address__
            - regex: (.+)
              replacement: /api/v1/nodes/$$1/proxy/metrics
              source_labels:
              - __meta_kubernetes_node_name
              target_label: __metrics_path__
            scheme: https
            tls_config:
              ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
              insecure_skip_verify: true

          - job_name: kubernetes-nodes-cadvisor
            bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
            kubernetes_sd_configs:
            - role: node
            relabel_configs:
            - action: labelmap
              regex: __meta_kubernetes_node_label_(.+)
            - replacement: kubernetes.default.svc:443
              target_label: __address__
            - regex: (.+)
              replacement: /api/v1/nodes/$$1/proxy/metrics/cadvisor
              source_labels:
              - __meta_kubernetes_node_name
              target_label: __metrics_path__
            scheme: https
            tls_config:
              ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
              insecure_skip_verify: true

          - job_name: kubernetes-service-endpoints
            kubernetes_sd_configs:
            - role: endpoints
            relabel_configs:
            - action: keep
              regex: true
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_scrape
            - action: replace
              regex: (https?)
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_scheme
              target_label: __scheme__
            - action: replace
              regex: (.+)
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_path
              target_label: __metrics_path__
            - action: replace
              regex: ([^:]+)(?::\d+)?;(\d+)
              replacement: $$1:$$2
              source_labels:
              - __address__
              - __meta_kubernetes_service_annotation_prometheus_io_port
              target_label: __address__
            - action: labelmap
              regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
              replacement: __param_$$1
            - action: labelmap
              regex: __meta_kubernetes_service_label_(.+)
            - action: replace
              source_labels:
              - __meta_kubernetes_namespace
              target_label: kubernetes_namespace
            - action: replace
              source_labels:
              - __meta_kubernetes_service_name
              target_label: kubernetes_name
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_node_name
              target_label: kubernetes_node

          - job_name: kubernetes-service-endpoints-slow
            kubernetes_sd_configs:
            - role: endpoints
            relabel_configs:
            - action: keep
              regex: true
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
            - action: replace
              regex: (https?)
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_scheme
              target_label: __scheme__
            - action: replace
              regex: (.+)
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_path
              target_label: __metrics_path__
            - action: replace
              regex: ([^:]+)(?::\d+)?;(\d+)
              replacement: $$1:$$2
              source_labels:
              - __address__
              - __meta_kubernetes_service_annotation_prometheus_io_port
              target_label: __address__
            - action: labelmap
              regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
              replacement: __param_$$1
            - action: labelmap
              regex: __meta_kubernetes_service_label_(.+)
            - action: replace
              source_labels:
              - __meta_kubernetes_namespace
              target_label: kubernetes_namespace
            - action: replace
              source_labels:
              - __meta_kubernetes_service_name
              target_label: kubernetes_name
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_node_name
              target_label: kubernetes_node
            scrape_interval: 5m
            scrape_timeout: 30s

          - job_name: prometheus-pushgateway
            honor_labels: true
            kubernetes_sd_configs:
            - role: service
            relabel_configs:
            - action: keep
              regex: pushgateway
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_probe

          - job_name: kubernetes-services
            kubernetes_sd_configs:
            - role: service
            metrics_path: /probe
            params:
              module:
              - http_2xx
            relabel_configs:
            - action: keep
              regex: true
              source_labels:
              - __meta_kubernetes_service_annotation_prometheus_io_probe
            - source_labels:
              - __address__
              target_label: __param_target
            - replacement: blackbox
              target_label: __address__
            - source_labels:
              - __param_target
              target_label: instance
            - action: labelmap
              regex: __meta_kubernetes_service_label_(.+)
            - source_labels:
              - __meta_kubernetes_namespace
              target_label: kubernetes_namespace
            - source_labels:
              - __meta_kubernetes_service_name
              target_label: kubernetes_name

          - job_name: kubernetes-pods
            kubernetes_sd_configs:
            - role: pod
            relabel_configs:
            - action: keep
              regex: true
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_scrape
            - action: replace
              regex: (https?)
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_scheme
              target_label: __scheme__
            - action: replace
              regex: (.+)
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_path
              target_label: __metrics_path__
            - action: replace
              regex: ([^:]+)(?::\d+)?;(\d+)
              replacement: $$1:$$2
              source_labels:
              - __address__
              - __meta_kubernetes_pod_annotation_prometheus_io_port
              target_label: __address__
            - action: labelmap
              regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
              replacement: __param_$$1
            - action: labelmap
              regex: __meta_kubernetes_pod_label_(.+)
            - action: replace
              source_labels:
              - __meta_kubernetes_namespace
              target_label: kubernetes_namespace
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_name
              target_label: kubernetes_pod_name
            - action: drop
              regex: Pending|Succeeded|Failed|Completed
              source_labels:
              - __meta_kubernetes_pod_phase

          - job_name: kubernetes-pods-slow
            scrape_interval: 5m
            scrape_timeout: 30s
            kubernetes_sd_configs:
            - role: pod
            relabel_configs:
            - action: keep
              regex: true
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
            - action: replace
              regex: (https?)
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_scheme
              target_label: __scheme__
            - action: replace
              regex: (.+)
              source_labels:
              - __meta_kubernetes_pod_annotation_prometheus_io_path
              target_label: __metrics_path__
            - action: replace
              regex: ([^:]+)(?::\d+)?;(\d+)
              replacement: $$1:$$2
              source_labels:
              - __address__
              - __meta_kubernetes_pod_annotation_prometheus_io_port
              target_label: __address__
            - action: labelmap
              regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
              replacement: __param_$1
            - action: labelmap
              regex: __meta_kubernetes_pod_label_(.+)
            - action: replace
              source_labels:
              - __meta_kubernetes_namespace
              target_label: namespace
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_name
              target_label: pod
            - action: drop
              regex: Pending|Succeeded|Failed|Completed
              source_labels:
              - __meta_kubernetes_pod_phase

    processors:
      batch/metrics:
        timeout: 60s

    exporters:
      prometheusremotewrite:
        endpoint: "http://kube-prometheus-stack-prometheus:9090/api/v1/write"
        # curl -XPOST kube-prometheus-stack-prometheus:9090/api/v1/write

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: [batch/metrics]
          exporters: [prometheusremotewrite]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-prometheus-role
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs:
      - /metrics
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-prometheus-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-prometheus-role
subjects:
  - kind: ServiceAccount
    name: amp-iamproxy-ingest-role
    namespace: monitoring

EOF

# 이제 배포가 완료 되었고 포드들의 상태를 확인 해본다. 
(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 adot]# kubectl get po -n monitoring
NAME                                              READY   STATUS    RESTARTS   AGE
# metric 을 수집하기 위한 ethtools exporter 
ethtool-exporter-62k4m                            1/1     Running   0          10h
ethtool-exporter-f4ktg                            1/1     Running   0          10h
ethtool-exporter-v4btb                            1/1     Running   0          10h
# prom stack : prometheus, grafana 만 구성했다. 
kube-prometheus-stack-grafana-7745c6cdd7-bjwvp    3/3     Running   0          21m
kube-prometheus-stack-operator-75b7b9747d-c5sqr   1/1     Running   0          21m
prometheus-kube-prometheus-stack-prometheus-0     2/2     Running   0          21m
# otel collector pod 
observability-collector-649f8ccfbb-9zw5n          1/1     Running   0          18m

(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 adot]# kubectl get po -n opentelemetry-operator-system
NAME                                      READY   STATUS    RESTARTS   AGE
opentelemetry-operator-5878dff984-v8zkf   2/2     Running   0          10h

# otel collector 에서는 로그가 다음과 같이 출력 된다. 
(se4ofnight@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 adot]# kubectl logs -n monitoring observability-collector-649f8ccfbb-9zw5n
2023/05/09 02:47:56 AWS OTel Collector version: v0.20.0
2023/05/09 02:47:56 found no extra config, skip it, err: open /opt/aws/aws-otel-collector/etc/extracfg.txt: no such file or directory
2023-05-09T02:47:56.303Z	info	service/telemetry.go:102	Setting up own telemetry...
2023-05-09T02:47:56.303Z	info	service/telemetry.go:137	Serving Prometheus metrics	{"address": ":8888", "level": "basic"}
2023-05-09T02:47:56.304Z	info	extensions/extensions.go:42	Starting extensions...
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:74	Starting exporters...
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:78	Exporter is starting...	{"kind": "exporter", "data_type": "metrics", "name": "prometheusremotewrite"}
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:82	Exporter started.	{"kind": "exporter", "data_type": "metrics", "name": "prometheusremotewrite"}
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:86	Starting processors...
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:90	Processor is starting...	{"kind": "processor", "name": "batch/metrics", "pipeline": "metrics"}
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:94	Processor started.	{"kind": "processor", "name": "batch/metrics", "pipeline": "metrics"}
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:98	Starting receivers...
2023-05-09T02:47:56.304Z	info	pipelines/pipelines.go:102	Receiver is starting...	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics"}
2023-05-09T02:47:56.304Z	info	kubernetes/kubernetes.go:326	Using pod service account via in-cluster config	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics", "discovery": "kubernetes"}
2023-05-09T02:47:56.305Z	info	kubernetes/kubernetes.go:326	Using pod service account via in-cluster config	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics", "discovery": "kubernetes"}
2023-05-09T02:47:56.305Z	info	kubernetes/kubernetes.go:326	Using pod service account via in-cluster config	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics", "discovery": "kubernetes"}
2023-05-09T02:47:56.305Z	info	kubernetes/kubernetes.go:326	Using pod service account via in-cluster config	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics", "discovery": "kubernetes"}
2023-05-09T02:47:56.306Z	info	pipelines/pipelines.go:106	Receiver started.	{"kind": "receiver", "name": "prometheus", "pipeline": "metrics"}
2023-05-09T02:47:56.306Z	info	service/collector.go:215	Starting aws-otel-collector...	{"Version": "v0.20.0", "NumCPU": 2}
2023-05-09T02:47:56.306Z	info	service/collector.go:128	Everything is ready. Begin running and processing data.

```


![실행 결과](/posts/study/cloudnet-aews/2w/images/result.png)