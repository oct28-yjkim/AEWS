---
title: "EKS 의 Storage, NodeGroup 를 알아봅시다. "
date: 2023-05-14T08:15:25+09:00
description: ""
menu:
  sidebar:
    name: 3w
    identifier: 3w
    parent: cloudnet-aews
    weight: 30
author:
  name: john doe
  image: /images/author/john.png
math: true
hero: images/forest.jpg
---


## 개요 

3주차 세션 Storage, NodeGroup 관련 내용이다. 

network, storage 관련 되어서는 구축 후에 정상 동작 하지 않는 케이스도 상당히 많아서 
기존에 작업 했을떄에는 최초 1회 동작 검증까지도 병행 했던 기억들이 있다. 
나름 잘 구성 하였는데도 엔지니어의 미스 혹은 driver 혹은 구성 미스 등이 많기 떄문에 잘 확인 하기 위한 
잘 정리 해놓는것이 중요 하다고 생각한다.

이번주 주제는 지난 PKOS 에서 잘 안됬던 부분에 대한 복습 위주로 진행 해보는게 좋겠다.
* AWS EBS Controller 
* AWS Volume Snapshot 을 이용하여 snapshot 생성 
* 도전과제 진행 

---

## AWS EBS Controller 

###  Volume Snapshots controller 설명 

![Volume-snapshot-controller](/posts/study/cloudnet-aews/3w/images/volume-controller.png)

1. API 서버를 통해 PVC 를 생성
2. csi-controller 에서 PVC filter 기반으로 watch 후 AWS로 EBS 를 생성 
3. AWS 에서는 요청 받은 EBS 를 생성 후 pod 의 device 에 mount 
4. API 서버에서는 pod 에게 mount 요청 
5. kubelet 에서는 해당 workload pod 에서 mount 요청
6. workload pod 에서 전달받은 ebs device 를 마운트 하여 사용 된다는 이야기를 하고 있다. 

### 작업의 순서

1. eks addon 
2. iam policy
3. csi driver, csi node 확인 
4. sc 생성 
5. pv,pvc 확인 


```sh
# 현재 사용중인 기본 addon version 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws eks describe-addon-versions \
>     --addon-name aws-ebs-csi-driver \
>     --kubernetes-version 1.24 \
>     --query "addons[].addonVersions[].[addonVersion, compatibilities[].defaultVersion]" \
>     --output text
v1.18.0-eksbuild.1
True
v1.17.0-eksbuild.1
False

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# echo $CLUSTER_NAME
yjkim-eks

# IRSA 관리형 정책 사용 : AmazonEBSCSIDriverPolicy
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ${CLUSTER_NAME} \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get iamserviceaccount --cluster yjkim-eks
NAMESPACE	NAME				ROLE ARN
kube-system	aws-load-balancer-controller	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-VT1KXYL24RY5
kube-system	ebs-csi-controller-sa		arn:aws:iam::123123123:role/AmazonEKS_EBS_CSI_DriverRole

# EBS CSI driver addon 추가 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl create addon --name aws-ebs-csi-driver --cluster ${CLUSTER_NAME} --service-account-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get addon --cluster ${CLUSTER_NAME}
2023-05-14 15:53:03 [ℹ]  Kubernetes version "1.24" in use by cluster "yjkim-eks"
2023-05-14 15:53:03 [ℹ]  getting all addons
2023-05-14 15:53:04 [ℹ]  to see issues for an addon run `eksctl get addon --name <addon-name> --cluster <cluster-name>`
NAME			VERSION			STATUS		ISSUES	IAMROLE									UPDATE AVAILABLE	CONFIGURATION VALUES
aws-ebs-csi-driver	v1.18.0-eksbuild.1	CREATING	0	arn:aws:iam::123123123:role/AmazonEKS_EBS_CSI_DriverRole
coredns			v1.9.3-eksbuild.3	ACTIVE		0
kube-proxy		v1.24.10-eksbuild.2	ACTIVE		0
vpc-cni			v1.12.6-eksbuild.2	ACTIVE		0	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-vpc-cni-Role1-1UZNP7WW6JI89

# addon 을 추가하면 csi pod 가 추가 된다. 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get deploy,ds -l=app.kubernetes.io/name=aws-ebs-csi-driver -n kube-system
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ebs-csi-controller   2/2     2            2           69s

NAME                                  DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
daemonset.apps/ebs-csi-node           3         3         3       3            3           kubernetes.io/os=linux     69s
daemonset.apps/ebs-csi-node-windows   0         0         0       0            0           kubernetes.io/os=windows   69s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -l 'app in (ebs-csi-controller,ebs-csi-node)'
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-78d7977c7f-lfmnw   6/6     Running   0          70s
ebs-csi-controller-78d7977c7f-q426t   6/6     Running   0          70s
ebs-csi-node-57z9z                    3/3     Running   0          70s
ebs-csi-node-8zccg                    3/3     Running   0          70s
ebs-csi-node-zczm9                    3/3     Running   0          70s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -l app.kubernetes.io/component=csi-driver
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-78d7977c7f-lfmnw   6/6     Running   0          71s
ebs-csi-controller-78d7977c7f-q426t   6/6     Running   0          71s
ebs-csi-node-57z9z                    3/3     Running   0          71s
ebs-csi-node-8zccg                    3/3     Running   0          71s
ebs-csi-node-zczm9                    3/3     Running   0          71s

# csi spec 에 따른 csi container 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -l app=ebs-csi-controller -o jsonpath='{.items[0].spec.containers[*].name}' ; echo
ebs-plugin csi-provisioner csi-attacher csi-snapshotter csi-resizer liveness-probe

# gp3 sc 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  89m

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat <<EOT > gp3-sc.yaml
> kind: StorageClass
> apiVersion: storage.k8s.io/v1
> metadata:
>   name: gp3
> allowVolumeExpansion: true
> provisioner: ebs.csi.aws.com
> volumeBindingMode: WaitForFirstConsumer
> parameters:
>   type: gp3
>   allowAutoIOPSPerGBIncrease: 'true'
>   encrypted: 'true'
> EOT
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f gp3-sc.yaml

storageclass.storage.k8s.io/gp3 created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  89m
gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   1s

# pvc, pv 생성후 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat <<EOT > awsebs-pvc.yaml
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: ebs-claim
> spec:
>   accessModes:
>     - ReadWriteOnce
>   resources:
>     requests:
>       storage: 4Gi
>   storageClassName: gp3
> EOT

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f awsebs-pvc.yaml
persistentvolumeclaim/ebs-claim created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pvc,pv
NAME                              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ebs-claim   Pending                                      gp3            1s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat <<EOT > awsebs-pod.yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: app
> spec:
>   terminationGracePeriodSeconds: 3
>   containers:
>   - name: app
>     image: centos
>     command: ["/bin/sh"]
>     args: ["-c", "while true; do echo \$(date -u) >> /data/out.txt; sleep 5; done"]
>     volumeMounts:
>     - name: persistent-storage
>       mountPath: /data
>   volumes:
>   - name: persistent-storage
>     persistentVolumeClaim:
>       claimName: ebs-claim
> EOT

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f awsebs-pod.yaml
pod/app created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f awsebs-pod.yaml
pod/app created
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pvc,pv,pod
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ebs-claim   Bound    pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134   4Gi        RWO            gp3            46s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
persistentvolume/pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134   4Gi        RWO            Delete           Bound    default/ebs-claim   gp3                     40s

NAME      READY   STATUS    RESTARTS   AGE
pod/app   1/1     Running   0          43s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get VolumeAttachment
NAME                                                                   ATTACHER          PV                                         NODE                                               ATTACHED   AGE
csi-fb53453f430cff2f1f93bfbfbe7c210b5593750a646b8f74f64d135268a9cbe0   ebs.csi.aws.com   pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134   ip-192-168-2-218.ap-northeast-3.compute.internal   true       41s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws ec2 describe-volumes --volume-ids $(kubectl get pv -o jsonpath="{.items[0].spec.csi.volumeHandle}") | jq
{
  "Volumes": [
    {
      "Attachments": [
        {
          "AttachTime": "2023-05-14T06:57:30+00:00",
          "Device": "/dev/xvdaa",
          "InstanceId": "i-0042cb2d53c422572",
          "State": "attached",
          "VolumeId": "vol-0e3c276cd92415a98",
          "DeleteOnTermination": false
        }
      ],
      "AvailabilityZone": "ap-northeast-3b",
      "CreateTime": "2023-05-14T06:57:25.704000+00:00",
      "Encrypted": true,
      "KmsKeyId": "arn:aws:kms:ap-northeast-3:123123123:key/685570ad-29d8-48d0-8606-19324101ca0c",
      "Size": 4,
      "SnapshotId": "",
      "State": "in-use",
      "VolumeId": "vol-0e3c276cd92415a98",
      "Iops": 3000,
      "Tags": [
        {
          "Key": "kubernetes.io/cluster/yjkim-eks",
          "Value": "owned"
        },
        {
          "Key": "CSIVolumeName",
          "Value": "pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134"
        },
        {
          "Key": "Name",
          "Value": "yjkim-eks-dynamic-pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134"
        },
        {
          "Key": "ebs.csi.aws.com/cluster",
          "Value": "true"
        },
        {
          "Key": "kubernetes.io/created-for/pvc/name",
          "Value": "ebs-claim"
        },
        {
          "Key": "KubernetesCluster",
          "Value": "yjkim-eks"
        },
        {
          "Key": "kubernetes.io/created-for/pv/name",
          "Value": "pvc-8f72e5ac-7f28-4bf9-92da-e734a5664134"
        },
        {
          "Key": "kubernetes.io/created-for/pvc/namespace",
          "Value": "default"
        }
      ],
      "VolumeType": "gp3",
      "MultiAttachEnabled": false,
      "Throughput": 125
    }
  ]
}

# 정상적으로 잘 write 하고 있는것을 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl exec app -- tail -f /data/out.txt
Sun May 14 06:59:07 UTC 2023
Sun May 14 06:59:12 UTC 2023
Sun May 14 06:59:17 UTC 2023

```


---

## AWS Volume Snapshots Controller

### 작업 순서 

1. CRD 생성 
2. Controller 생성 
3. SnapshotClass 생성 
4. 위에서 생성한 test pv로 snapshot 생성 확인 

```sh 

# crd 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f snapshot.storage.k8s.io_volumesnapshots.yaml,snapshot.storage.k8s.io_volumesnapshotclasses.yaml,snapshot.storage.k8s.io_volumesnapshotcontents.yaml
customresourcedefinition.apiextensions.k8s.io/volumesnapshots.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshotclasses.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshotcontents.snapshot.storage.k8s.io created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get crd | grep snapshot
volumesnapshotclasses.snapshot.storage.k8s.io    2023-05-14T07:04:13Z
volumesnapshotcontents.snapshot.storage.k8s.io   2023-05-14T07:04:13Z
volumesnapshots.snapshot.storage.k8s.io          2023-05-14T07:04:13Z

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl api-resources  | grep snapshot
volumesnapshotclasses             vsclass,vsclasses   snapshot.storage.k8s.io/v1             false        VolumeSnapshotClass
volumesnapshotcontents            vsc,vscs            snapshot.storage.k8s.io/v1             false        VolumeSnapshotContent
volumesnapshots                   vs                  snapshot.storage.k8s.io/v1             true         VolumeSnapshot

# controller 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f rbac-snapshot-controller.yaml,setup-snapshot-controller.yaml
serviceaccount/snapshot-controller created
clusterrole.rbac.authorization.k8s.io/snapshot-controller-runner created
clusterrolebinding.rbac.authorization.k8s.io/snapshot-controller-role created
role.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
rolebinding.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
deployment.apps/snapshot-controller created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get deploy -n kube-system snapshot-controller
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
snapshot-controller   0/2     2            0           0s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -l app=snapshot-controller
NAME                                   READY   STATUS              RESTARTS   AGE
snapshot-controller-76494bf6c9-6r4lk   0/1     ContainerCreating   0          2s
snapshot-controller-76494bf6c9-qhc8p   0/1     ContainerCreating   0          2s

# snapshots class 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/snapshot/manifests/classes/snapshotclass.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f snapshotclass.yaml
volumesnapshotclass.snapshot.storage.k8s.io/csi-aws-vsc created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get vsclass # or volumesnapshotclasses
NAME          DRIVER            DELETIONPOLICY   AGE
csi-aws-vsc   ebs.csi.aws.com   Delete           2s

# volume snapshot 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/ebs-volume-snapshot.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat ebs-volume-snapshot.yaml | yh
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ebs-volume-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: ebs-claim

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f ebs-volume-snapshot.yaml
volumesnapshot.snapshot.storage.k8s.io/ebs-volume-snapshot created

# 생성한 snapshot 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get volumesnapshot
NAME                  READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS   SNAPSHOTCONTENT                                    CREATIONTIME   AGE
ebs-volume-snapshot   true         ebs-claim                           4Gi           csi-aws-vsc     snapcontent-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c   46s            46s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get volumesnapshot ebs-volume-snapshot -o jsonpath={.status.boundVolumeSnapshotContentName} ; echo
snapcontent-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get volumesnapshotcontents
NAME                                               READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER            VOLUMESNAPSHOTCLASS   VOLUMESNAPSHOT        VOLUMESNAPSHOTNAMESPACE   AGE
snapcontent-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c   true         4294967296    Delete           ebs.csi.aws.com   csi-aws-vsc           ebs-volume-snapshot   default                   49s


(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws ec2 describe-snapshots --owner-ids self | jq
{
  "Snapshots": [
    {
      "Description": "Created by AWS EBS CSI driver for volume vol-0e3c276cd92415a98",
      "Encrypted": true,
      "KmsKeyId": "arn:aws:kms:ap-northeast-3:123123123:key/685570ad-29d8-48d0-8606-19324101ca0c",
      "OwnerId": "123123123",
      "Progress": "100%",
      "SnapshotId": "snap-04067400af22e7008",
      "StartTime": "2023-05-14T07:07:46.755000+00:00",
      "State": "completed",
      "VolumeId": "vol-0e3c276cd92415a98",
      "VolumeSize": 4,
      "Tags": [
        {
          "Key": "Name",
          "Value": "yjkim-eks-dynamic-snapshot-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c"
        },
        {
          "Key": "ebs.csi.aws.com/cluster",
          "Value": "true"
        },
        {
          "Key": "CSIVolumeSnapshotName",
          "Value": "snapshot-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c"
        },
        {
          "Key": "kubernetes.io/cluster/yjkim-eks",
          "Value": "owned"
        }
      ],
      "StorageTier": "standard"
    }
  ]
}
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws ec2 describe-snapshots --owner-ids self --query 'Snapshots[]' --output table
----------------------------------------------------------------------------------------------------------
|                                            DescribeSnapshots                                           |
+--------------+-----------------------------------------------------------------------------------------+
|  Description |  Created by AWS EBS CSI driver for volume vol-0e3c276cd92415a98                         |
|  Encrypted   |  True                                                                                   |
|  KmsKeyId    |  arn:aws:kms:ap-northeast-3:123123123:key/685570ad-29d8-48d0-8606-19324101ca0c       |
|  OwnerId     |  123123123                                                                           |
|  Progress    |  100%                                                                                   |
|  SnapshotId  |  snap-04067400af22e7008                                                                 |
|  StartTime   |  2023-05-14T07:07:46.755000+00:00                                                       |
|  State       |  completed                                                                              |
|  StorageTier |  standard                                                                               |
|  VolumeId    |  vol-0e3c276cd92415a98                                                                  |
|  VolumeSize  |  4                                                                                      |
+--------------+-----------------------------------------------------------------------------------------+
||                                                 Tags                                                 ||
|+----------------------------------+-------------------------------------------------------------------+|
||                Key               |                               Value                               ||
|+----------------------------------+-------------------------------------------------------------------+|
||  Name                            |  yjkim-eks-dynamic-snapshot-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c  ||
||  ebs.csi.aws.com/cluster         |  true                                                             ||
||  CSIVolumeSnapshotName           |  snapshot-e2bbed3c-3c5f-4640-8d32-35e3aa18e42c                    ||
||  kubernetes.io/cluster/yjkim-eks |  owned                                                            ||
|+----------------------------------+-------------------------------------------------------------------+|

# 강제로 날려버리기 
kubectl delete pod app && kubectl delete pvc ebs-claim

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat ebs-snapshot-restored-claim.yaml | yh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-snapshot-restored-claim
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  dataSource:
    name: ebs-volume-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f ebs-snapshot-restored-claim.yaml
persistentvolumeclaim/ebs-snapshot-restored-claim created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pvc,pv
NAME                                                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ebs-snapshot-restored-claim   Pending                                      gp3            33s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/ebs-snapshot-restored-pod.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cat ebs-snapshot-restored-pod.yaml | yh
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-snapshot-restored-claim

# 생성을 해야 pv 의 pending 이 변경 된다. 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl apply -f ebs-snapshot-restored-pod.yaml
pod/app created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pvc,pv
NAME                                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ebs-snapshot-restored-claim   Bound    pvc-7a303cbe-28d2-40e1-81b5-50e8e57fd31a   4Gi        RWO            gp3            66s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
persistentvolume/pvc-7a303cbe-28d2-40e1-81b5-50e8e57fd31a   4Gi        RWO            Delete           Bound    default/ebs-snapshot-restored-claim   gp3                     6s
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl exec app -- cat /data/out.txt
Sun May 14 06:57:42 UTC 2023
Sun May 14 06:57:47 UTC 2023

```

---

## Volume Snapshots Scheduler

### 작업 순서 

1. Volume snapshots Scheduler 설치 
2. sample pv,pvc,po 설치 
3. scheduler manifest 생성 
4. 확인 

```sh 
# 설치 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# helm repo add backube https://backube.github.io/helm-charts/
"backube" has been added to your repositories
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl create namespace backube-snapscheduler
namespace/backube-snapscheduler created
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# helm install -n backube-snapscheduler snapscheduler backube/snapscheduler

NAME: snapscheduler
LAST DEPLOYED: Sun May 14 17:48:54 2023
NAMESPACE: backube-snapscheduler
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing SnapScheduler!

Chart version: 3.2.0
SnapScheduler version: 3.2.0

The SnapScheduler operator is now installed in the backube-snapscheduler
namespace, and snapshotschedules should be enabled cluster-wide.

See https://backube.github.io/snapscheduler/usage.html to get started.

Schedules can be viewed via:
$ kubectl -n <mynampspace> get snapshotschedules

# sample workload deploy 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat << EOF | kubectl create -f -
> ---
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: ebs-claim
> spec:
>   accessModes:
>     - ReadWriteOnce
>   resources:
>     requests:
>       storage: 4Gi
>   storageClassName: gp3
> ---
> apiVersion: v1
> kind: Pod
> metadata:
>   name: app
> spec:
>   terminationGracePeriodSeconds: 3
>   containers:
>   - name: app
>     image: centos
>     command: ["/bin/sh"]
>     args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
>     volumeMounts:
>     - name: persistent-storage
>       mountPath: /data
>   volumes:
>   - name: persistent-storage
>     persistentVolumeClaim:
>       claimName: ebs-claim
> EOF
persistentvolumeclaim/ebs-claim created
pod/app created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat << EOF | kubectl create -f -
> apiVersion: snapscheduler.backube/v1
> kind: SnapshotSchedule
> metadata:
>   name: every-5min
>   namespace: default
> spec:
>   claimSelector:  # optional
>   disabled: false  # optional
>   retention:
>     expires: "24h"  # optional
>     maxCount: 10  # optional
>   schedule: "*/5 * * * *"
>   snapshotTemplate:
>     labels:  # optional
>       mylabel: myvalue
>     snapshotClassName: csi-aws-vsc  # optional
>
> EOF
snapshotschedule.snapscheduler.backube/every-5min created

# snapshot 생성 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl get snapshotschedules
NAME         SCHEDULE      MAX AGE   MAX NUM   DISABLED   NEXT SNAPSHOT
every-5min   */5 * * * *   24h       10        false      2023-05-14T08:55:00Z


```

---

## Velero Backup : 작성중

* 다음에.. : https://aws.amazon.com/ko/blogs/containers/backup-and-restore-your-amazon-eks-cluster-resources-using-velero/

---

## AWS EFS Controller 

![efs-controller](/posts/study/cloudnet-aews/3w/images/efs-controller.png)

### 작업순서 

1. 앞서 생성한 efs 정보 확인 
2. efs controller 에서 사용할 iam 정책 IRSA 생성 
3. EFS Controller 설치 
4. EFS file system 을 다수 파드가 static 하게 사용 
5. EFS file system 을 다수 파드가 dynamic 하게 사용 

### EFS Controller 설치 

```sh 

# EFS 정보 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
fs-0882e651f234e9c01

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws iam create-policy --policy-name AmazonEKS_EFS_CSI_Driver_Policy --policy-document file://iam-policy-example.json
{
    "Policy": {
        "PolicyName": "AmazonEKS_EFS_CSI_Driver_Policy",
        "PolicyId": "ANPAXFXFAAOJSNBXVBN6R",
        "Arn": "arn:aws:iam::123123123:policy/AmazonEKS_EFS_CSI_Driver_Policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-05-14T07:25:21+00:00",
        "UpdateDate": "2023-05-14T07:25:21+00:00"
    }
}

# IRSA 설정 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl create iamserviceaccount \
>   --name efs-csi-controller-sa \
>   --namespace kube-system \
>   --cluster ${CLUSTER_NAME} \
>   --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
>   --approve
2023-05-14 16:25:56 [ℹ]  2 existing iamserviceaccount(s) (kube-system/aws-load-balancer-controller,kube-system/ebs-csi-controller-sa) will be excluded
2023-05-14 16:25:56 [ℹ]  1 iamserviceaccount (kube-system/efs-csi-controller-sa) was included (based on the include/exclude rules)
2023-05-14 16:25:56 [!]  serviceaccounts that exist in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
2023-05-14 16:25:56 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/efs-csi-controller-sa",
        create serviceaccount "kube-system/efs-csi-controller-sa",
    } }2023-05-14 16:25:56 [ℹ]  building iamserviceaccount stack "eksctl-yjkim-eks-addon-iamserviceaccount-kube-system-efs-csi-controller-sa"
2023-05-14 16:25:56 [ℹ]  deploying stack "eksctl-yjkim-eks-addon-iamserviceaccount-kube-system-efs-csi-controller-sa"
2023-05-14 16:25:56 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-addon-iamserviceaccount-kube-system-efs-csi-controller-sa"
2023-05-14 16:26:26 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-addon-iamserviceaccount-kube-system-efs-csi-controller-sa"
2023-05-14 16:26:26 [ℹ]  created serviceaccount "kube-system/efs-csi-controller-sa"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get sa -n kube-system efs-csi-controller-sa -o yaml | head -5
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-1PFCU3SMF1MRR

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# eksctl get iamserviceaccount --cluster yjkim-eks
NAMESPACE	NAME				ROLE ARN
kube-system	aws-load-balancer-controller	arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-VT1KXYL24RY5
kube-system	ebs-csi-controller-sa		arn:aws:iam::123123123:role/AmazonEKS_EBS_CSI_DriverRole
kube-system	efs-csi-controller-sa		arn:aws:iam::123123123:role/eksctl-yjkim-eks-addon-iamserviceaccount-kub-Role1-1PFCU3SMF1MRR

# helm chart 를 이용해 설치 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
"aws-efs-csi-driver" has been added to your repositories

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "aws-efs-csi-driver" chart repository
Update Complete. ⎈Happy Helming!⎈

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
>     --namespace kube-system \
>     --set image.repository=602401143452.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/eks/aws-efs-csi-driver \
>     --set controller.serviceAccount.create=false \
>     --set controller.serviceAccount.name=efs-csi-controller-sa
Release "aws-efs-csi-driver" does not exist. Installing it now.
NAME: aws-efs-csi-driver
LAST DEPLOYED: Sun May 14 16:28:06 2023
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To verify that aws-efs-csi-driver has started, run:

    kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# helm list -n kube-system
NAME              	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                   	APP VERSION
aws-efs-csi-driver	kube-system	1       	2023-05-14 16:28:06.894304587 +0900 KST	deployed	aws-efs-csi-driver-2.4.3	1.5.5

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
NAME                                 READY   STATUS    RESTARTS   AGE
efs-csi-controller-74756bf58-8ctpr   3/3     Running   0          14s
efs-csi-controller-74756bf58-k9hvk   3/3     Running   0          14s
efs-csi-node-5ngdk                   3/3     Running   0          14s
efs-csi-node-6dxnl                   3/3     Running   0          14s
efs-csi-node-vnmhl                   3/3     Running   0          14s

# AWS Console > EFS > 파일시스템 > 네트워크탭 > 탑재대상 ID 로 확인 가능 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# aws efs describe-mount-targets     --mount-target-id  fsmt-00de5f3c46447f30a
{
    "MountTargets": [
        {
            "OwnerId": "123123123",
            "MountTargetId": "fsmt-00de5f3c46447f30a",
            "FileSystemId": "fs-0882e651f234e9c01",
            "SubnetId": "subnet-06b50e559f31ca9c3",
            "LifeCycleState": "available",
            "IpAddress": "192.168.1.38",
            "NetworkInterfaceId": "eni-0a395764c5a240923",
            "AvailabilityZoneId": "apne3-az3",
            "AvailabilityZoneName": "ap-northeast-3a",
            "VpcId": "vpc-08c825742b5216324"
        }
    ]
}

```

### Static Example 

```sh 
# 실습 코드 준비 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git /root/efs-csi
Cloning into '/root/efs-csi'...
remote: Enumerating objects: 16052, done.
remote: Counting objects: 100% (269/269), done.
remote: Compressing objects: 100% (185/185), done.
remote: Total 16052 (delta 65), reused 236 (delta 57), pack-reused 15783
Receiving objects: 100% (16052/16052), 16.90 MiB | 8.67 MiB/s, done.
Resolving deltas: 100% (7777/7777), done.
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 ~]# cd /root/efs-csi/examples/kubernetes/multiple_pods/specs && tree
.
├── claim.yaml
├── pod1.yaml
├── pod2.yaml
├── pv.yaml
└── storageclass.yaml

0 directories, 5 files

# SC 생성 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# cat storageclass.yaml | yh
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl apply -f storageclass.yaml
storageclass.storage.k8s.io/efs-sc created
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl get sc efs-sc
NAME     PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
efs-sc   efs.csi.aws.com   Delete          Immediate           false                  1s

# PV 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# EfsFsId=$(aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text)
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# sed -i "s/fs-4af69aab/$EfsFsId/g" pv.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# echo $EfsFsId
fs-0882e651f234e9c01

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# cat pv.yaml | yh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0882e651f234e9c01

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl apply -f claim.yaml
persistentvolumeclaim/efs-claim created

# Pod 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# cat claim.yaml | yh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# cat pod1.yaml pod2.yaml | yh
apiVersion: v1
kind: Pod
metadata:
  name: app1
spec:
  containers:
  - name: app1
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out1.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: efs-claim
apiVersion: v1
kind: Pod
metadata:
  name: app2
spec:
  containers:
  - name: app2
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out2.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: efs-claim
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl apply -f pod1.yaml,pod2.yaml
pod/app1 created
pod/app2 created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl exec -ti app1 -- sh -c "df -hT -t nfs4"
Filesystem           Type            Size      Used Available Use% Mounted on
127.0.0.1:/          nfs4            8.0E         0      8.0E   0% /data
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl exec -ti app2 -- sh -c "df -hT -t nfs4"
Filesystem           Type            Size      Used Available Use% Mounted on
127.0.0.1:/          nfs4            8.0E         0      8.0E   0% /data

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# tree /mnt/myefs
/mnt/myefs
├── out1.txt
└── out2.txt

0 directories, 2 files
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# tail -f /mnt/myefs/out1.txt
Sun May 14 07:51:51 UTC 2023
Sun May 14 07:51:56 UTC 2023

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 specs]# kubectl exec -ti app1 -- tail -f /data/out1.txt
Sun May 14 07:51:56 UTC 2023
Sun May 14 07:52:01 UTC 2023


```

### Dynamic Example 


```sh 
# storage class 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat storageclass.yaml | yh
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-92107410
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# sed -i "s/fs-92107410/$EfsFsId/g" storageclass.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl apply -f storageclass.yaml
storageclass.storage.k8s.io/efs-sc created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl get sc efs-sc
NAME     PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
efs-sc   efs.csi.aws.com   Delete          Immediate           false                  3s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0882e651f234e9c01
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional

# PVC, 파드 생성 및 확인 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat pod.yaml | yh
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl apply -f pod.yaml
persistentvolumeclaim/efs-claim created
pod/efs-app created

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl get pvc,pv,pod
NAME                                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ebs-snapshot-restored-claim   Bound    pvc-7a303cbe-28d2-40e1-81b5-50e8e57fd31a   4Gi        RWO            gp3            44m
persistentvolumeclaim/efs-claim                     Bound    pvc-6bc1dcb6-affd-45de-86cf-2755fe029ce1   5Gi        RWX            efs-sc         14s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
persistentvolume/pvc-6bc1dcb6-affd-45de-86cf-2755fe029ce1   5Gi        RWX            Delete           Bound    default/efs-claim                     efs-sc                  14s
persistentvolume/pvc-7a303cbe-28d2-40e1-81b5-50e8e57fd31a   4Gi        RWO            Delete           Bound    default/ebs-snapshot-restored-claim   gp3                     43m

NAME          READY   STATUS    RESTARTS   AGE
pod/efs-app   1/1     Running   0          14s

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl exec -it efs-app -- sh -c "df -hT -t nfs4"
Filesystem     Type  Size  Used Avail Use% Mounted on
127.0.0.1:/    nfs4  8.0E     0  8.0E   0% /data
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# tree /mnt/myefs
/mnt/myefs
├── dynamic_provisioning
│   └── pvc-6bc1dcb6-affd-45de-86cf-2755fe029ce1
│       └── out
├── out1.txt
└── out2.txt

2 directories, 3 files
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl exec efs-app -- bash -c "cat data/out"
Sun May 14 07:55:58 UTC 2023
Sun May 14 07:56:03 UTC 2023
Sun May 14 07:56:08 UTC 2023

# efs accesspoint 에서 조회 하면 efs 접속 포인트가 출력됨 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws efs describe-access-points
{
    "AccessPoints": [
        {
            "ClientToken": "pvc-6bc1dcb6-affd-45de-86cf-2755fe029ce1",
            "Tags": [
                {
                    "Key": "efs.csi.aws.com/cluster",
                    "Value": "true"
                }
            ],
            "AccessPointId": "fsap-09e43be7c4ff4029e",
            "AccessPointArn": "arn:aws:elasticfilesystem:ap-northeast-3:123123123:access-point/fsap-09e43be7c4ff4029e",
            "FileSystemId": "fs-0882e651f234e9c01",
            "PosixUser": {
                "Uid": 1000,
                "Gid": 1000
            },
            "RootDirectory": {
                "Path": "/dynamic_provisioning/pvc-6bc1dcb6-affd-45de-86cf-2755fe029ce1",
                "CreationInfo": {
                    "OwnerUid": 1000,
                    "OwnerGid": 1000,
                    "Permissions": "700"
                }
            },
            "OwnerId": "123123123",
            "LifeCycleState": "available"
        }
    ]
}

# 쿠버네티스 리소스 삭제
kubectl delete -f pod.yaml
kubectl delete -f storageclass.yaml

```

---

## Fargate : 작성중

![fargate](/posts/study/cloudnet-aews/3w/images/fargate1.png)

* Fargate 는 Serverless Dataplain 서비스 중에 하나로 노드 그룹이 AWS 에서 관리형으로 제공 해주는 서비스 이다. 
* 완전 관리형 서비스이기에 dataplain 을 scale 할 걱정할 필요가 없고 제공하는 pod 의 격리는 VM 수준의 격리를 제공 해준다고 한다. 
* 배포 하기 위해서는 fargateway Profile 생성이 필요 하다. 
  * 파드가 동작한 Private Subnet, 외부통신을 위한 nat gw 배치후 연결 해주어야한다. 
* 수동으로 작업 해주어야한다. 
 * AWS EKS Console > Computing -> Fargate Profile 추가 후 아래 작업
   * 이름 : yjkimfargate
   * subnet : PrivateSubnet1~3 
   * tag : Name / yjkimeks-fargate
   * ns : default, kube-system 
   * 역활 : 아래 있는 값으로 생성 AmazonEKSFargatePodExecutionRole

```sh 
# private subnet 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# PrivateSubnetRouteTable=$(aws ec2 describe-route-tables --filters Name=tag:Name,Values=$CLUSTER_NAME-PrivateSubnetRouteTable --query 'RouteTables[0].RouteTableId' --output text)
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# curl -s -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/natgw.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# sed -i "s/<PublicSubnet1>/$PrivateSubnet1/g" natgw.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# sed -i "s/<PrivateSubnetRouteTable>/$PrivateSubnetRouteTable/g" natgw.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat natgw.yaml
Resources:
# NAT Gateway
  MYVPCEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
  MYVPCNATGW:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId:
        Fn::GetAtt:
        - MYVPCEIP
        - AllocationId
      SubnetId: subnet-05cd05c8a742f6dbc
      Tags:
      - Key: Name
        Value: !Sub myeks-NATGW

# Add Route - PrivateSubnet RouteTable
  MYVPCNATGWRoute:
    Type: AWS::EC2::Route
    DependsOn: MYVPCNATGW
    Properties:
      RouteTableId: rtb-01226d2cbf370d715
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref MYVPCNATGW
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws cloudformation deploy --template-file natgw.yaml --stack-name $CLUSTER_NAME-natgw

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - yjkim-eks-natgw

# IAM role 등록 
cat << EOF >> pod-execution-role-trust-policy.json 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Condition": {
         "ArnLike": {
            "aws:SourceArn": "arn:aws:eks:ap-northeast-3:123123123:fargateprofile/yjkim-eks/*"
         }
      },
      "Principal": {
        "Service": "eks-fargate-pods.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF


aws iam create-role \
  --role-name AmazonEKSFargatePodExecutionRole \
  --assume-role-policy-document file://"pod-execution-role-trust-policy.json"

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy \
  --role-name AmazonEKSFargatePodExecutionRole

# console 에서 생성한 fargate 프로파일 조회 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# eksctl get fargateprofile --cluster $CLUSTER_NAME
NAME		SELECTOR_NAMESPACE	SELECTOR_LABELS	POD_EXECUTION_ROLE_ARN						SUBNETS					TAGS			STATUS
yjkimfargate	default			<none>		arn:aws:iam::123123123:role/AmazonEKSFargatePodExecutionRole	subnet-08db00b1422b35f9c,subnet-05cd05c8a742f6dbc,subnet-089fcb3e71ccac162	Name=yjkimeks-fargate	ACTIVE
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# eksctl get fargateprofile --cluster $CLUSTER_NAME -o json
[
    {
        "name": "yjkimfargate",
        "podExecutionRoleARN": "arn:aws:iam::123123123:role/AmazonEKSFargatePodExecutionRole",
        "selectors": [
            {
                "namespace": "default"
            }
        ],
        "subnets": [
            "subnet-08db00b1422b35f9c",
            "subnet-05cd05c8a742f6dbc",
            "subnet-089fcb3e71ccac162"
        ],
        "tags": {
            "Name": "yjkimeks-fargate"
        },
        "status": "ACTIVE"
    }
](yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# EKSSGID=$(aws eks describe-cluster --name $CLUSTER_NAME --query cluster.resourcesVpcConfig.clusterSeurityGroupId --output text)

# eksctl 에서 생성된 호스트와 접속할 수 있도록 보안그룹 룰 추가
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws ec2 authorize-security-group-ingress --group-id $EKSSGID --protocol '-1' --cidr 192.168.1.100/32
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0be76dcf71e9108e2",
            "GroupId": "sg-0cdda9223bf0f24f8",
            "GroupOwnerId": "123123123",
            "IsEgress": false,
            "IpProtocol": "-1",
            "FromPort": -1,
            "ToPort": -1,
            "CidrIpv4": "192.168.1.100/32"
        }
    ]
}

# fargate sample 앱 배포 
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/2/nginx-dp.yaml
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat nginx-dp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        node: fargate # 추가 해주어야함
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl apply -f nginx-dp.yaml
'deployment.apps/nginx-deployment created

# error, 다음에 하자
Events:
  Type     Reason           Age   From               Message
  ----     ------           ----  ----               -------
  Warning  LoggingDisabled  102s  fargate-scheduler  Disabled logging because aws-logging configmap was not found. configmap "aws-logging" not found

# 삭제 

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# kubectl delete -f nginx-dp.yaml
deployment.apps "nginx-deployment" deleted

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws eks delete-fargate-profile --cluster-name $CLUSTER_NAME --fargate-profile-name yjkimfargate
{
    "fargateProfile": {
        "fargateProfileName": "yjkimfargate",
        "fargateProfileArn": "arn:aws:eks:ap-northeast-3:123123123:fargateprofile/yjkim-eks/yjkimfargate/64c40cae-5c7b-7710-53cc-5ec63baa7826",
        "clusterName": "yjkim-eks",
        "createdAt": "2023-05-14T17:24:48.764000+09:00",
        "podExecutionRoleArn": "arn:aws:iam::123123123:role/AmazonEKSFargatePodExecutionRole",
        "subnets": [
            "subnet-08db00b1422b35f9c",
            "subnet-05cd05c8a742f6dbc",
            "subnet-089fcb3e71ccac162"
        ],
        "selectors": [
            {
                "namespace": "default",
                "labels": {}
            }
        ],
        "status": "DELETING",
        "tags": {
            "Name": "yjkimeks-fargate"
        }
    }
}

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws cloudformation delete-stack --stack-name yjkim-eks-natgw

```
---

## 노드 그룹 생성 및 instance store 생성

```sh 
# instance store 볼륨이 있는 c5 타입의 스토리지 크기 조회 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws ec2 describe-instance-types \
>  --filters "Name=instance-type,Values=c5*" "Name=instance-storage-supported,Values=true" \
>  --query "InstanceTypes[].[InstanceType, InstanceStorageInfo.TotalSizeInGB]" \
>  --output table
--------------------------
|  DescribeInstanceTypes |
+---------------+--------+
|  c5d.12xlarge |  1800  |
|  c5d.24xlarge |  3600  |
|  c5d.2xlarge  |  200   |
|  c5d.large    |  50    |
|  c5d.4xlarge  |  400   |
|  c5d.xlarge   |  100   |
|  c5d.18xlarge |  1800  |
|  c5d.metal    |  3600  |
|  c5d.9xlarge  |  900   |
+---------------+--------+

# 신규 노드 그룹 생성 
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# eksctl create nodegroup -c $CLUSTER_NAME -r $AWS_DEFAULT_REGION --subnet-ids "$PubSubnet1","$PubSubnet2","$PubSubnet3" --ssh-access \
>   -n ng2 -t c5d.large -N 1 -m 1 -M 1 --node-volume-size=30 --node-labels disk=nvme --max-pods-per-node 100 --dry-run > myng2.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat <<EOT > nvme.yaml
>   preBootstrapCommands:
>     - |
>       # Install Tools
>       yum install nvme-cli links tree jq tcpdump sysstat -y
>
>       # Filesystem & Mount
>       mkfs -t xfs /dev/nvme1n1
>       mkdir /data
>       mount /dev/nvme1n1 /data
>
>       # Get disk UUID
>       uuid=\$(blkid -o value -s UUID mount /dev/nvme1n1 /data)
>
>       # Mount the disk during a reboot
>       echo /dev/nvme1n1 /data xfs defaults,noatime 0 2 >> /etc/fstab
> EOT
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# sed -i -n -e '/volumeType/r nvme.yaml' -e '1,$p' myng2.yaml

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# cat myng2.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
managedNodeGroups:
- amiFamily: AmazonLinux2
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
  - subnet-06b50e559f31ca9c3
  - subnet-02ce02b6fbdede590
  - subnet-041fc554469da6a7f
  tags:
    alpha.eksctl.io/nodegroup-name: ng2
    alpha.eksctl.io/nodegroup-type: managed
  volumeIOPS: 3000
  volumeSize: 30
  volumeThroughput: 125
  volumeType: gp3
  preBootstrapCommands:
    - |
      # Install Tools
      yum install nvme-cli links tree jq tcpdump sysstat -y

      # Filesystem & Mount
      mkfs -t xfs /dev/nvme1n1
      mkdir /data
      mount /dev/nvme1n1 /data

      # Get disk UUID
      uuid=$(blkid -o value -s UUID mount /dev/nvme1n1 /data)

      # Mount the disk during a reboot
      echo /dev/nvme1n1 /data xfs defaults,noatime 0 2 >> /etc/fstab
metadata:
  name: yjkim-eks
  region: ap-northeast-3
  version: "1.24"

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# eksctl create nodegroup -f myng2.yaml
2023-05-14 17:58:43 [ℹ]  nodegroup "ng2" will use "" [AmazonLinux2/1.24]
2023-05-14 17:58:43 [ℹ]  using SSH public key "/root/.ssh/id_rsa.pub" as "eksctl-yjkim-eks-nodegroup-ng2-44:74:db:5a:9e:96:8e:0f:70:57:be:8f:83:28:e1:50"
2023-05-14 17:58:44 [ℹ]  1 existing nodegroup(s) (ng1) will be excluded
2023-05-14 17:58:44 [ℹ]  1 nodegroup (ng2) was included (based on the include/exclude rules)
2023-05-14 17:58:44 [ℹ]  will create a CloudFormation stack for each of 1 managed nodegroups in cluster "yjkim-eks"
2023-05-14 17:58:45 [ℹ]
2 sequential tasks: { fix cluster compatibility, 1 task: { 1 task: { create managed nodegroup "ng2" } }
}
2023-05-14 17:58:45 [ℹ]  checking cluster stack for missing resources
2023-05-14 17:58:45 [ℹ]  cluster stack has all required resources
2023-05-14 17:58:45 [ℹ]  building managed nodegroup stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 17:58:45 [ℹ]  deploying stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 17:58:45 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 17:59:16 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 18:00:10 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 18:01:52 [ℹ]  waiting for CloudFormation stack "eksctl-yjkim-eks-nodegroup-ng2"
2023-05-14 18:01:52 [ℹ]  no tasks
2023-05-14 18:01:52 [✔]  created 0 nodegroup(s) in cluster "yjkim-eks"
2023-05-14 18:01:52 [ℹ]  nodegroup "ng2" has 1 node(s)
2023-05-14 18:01:52 [ℹ]  node "ip-192-168-3-143.ap-northeast-3.compute.internal" is ready
2023-05-14 18:01:52 [ℹ]  waiting for at least 1 node(s) to become ready in "ng2"
2023-05-14 18:01:52 [ℹ]  nodegroup "ng2" has 1 node(s)
2023-05-14 18:01:52 [ℹ]  node "ip-192-168-3-143.ap-northeast-3.compute.internal" is ready
2023-05-14 18:01:52 [✔]  created 1 managed nodegroup(s) in cluster "yjkim-eks"
2023-05-14 18:01:52 [ℹ]  checking security group configuration for all nodegroups
2023-05-14 18:01:52 [ℹ]  all nodegroups have up-to-date cloudformation templates

# 보안그룹 확인 
(yjkim@yjkim-eks:\033[38;5;6mN/A) [root@yjkim-eks-bastion-EC2 test]# NG2SGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*ng2* --queoups[*].[GroupId]" --output text)
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# aws ec2 authorize-security-group-ingress --group-id $NG2SGID --protocol '-1' --cidr 192.168.1.100/32

{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0974b762de9adf5e2",
            "GroupId": "sg-0413fdc5af5f9351b",
            "GroupOwnerId": "123123123",
            "IsEgress": false,
            "IpProtocol": "-1",
            "FromPort": -1,
            "ToPort": -1,
            "CidrIpv4": "192.168.1.100/32"
        }
    ]
}

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@13.208.250.20 hostname
ip-192-168-3-143.ap-northeast-3.compute.internal

(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@$N4 sudo nvme list
Node             SN                   Model                                    Namespace Usage                      Format           FW Rev
---------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1     vol066eb6183f8bb3919 Amazon Elastic Block Store               1          32.21  GB /  32.21  GB    512   B +  0 B   1.0
/dev/nvme1n1     AWS25688E26FADF6B465 Amazon EC2 NVMe Instance Storage         1          50.00  GB /  50.00  GB    512   B +  0 B   0
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]#
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@$N4 sudo lsblk -e 7 -d
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
nvme1n1 259:0    0 46.6G  0 disk /data
nvme0n1 259:1    0   30G  0 disk
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@$N4 sudo df -hT -t xfs
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/nvme0n1p1 xfs    30G  3.3G   27G  11% /
/dev/nvme1n1   xfs    47G  365M   47G   1% /data
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@$N4 sudo tree /data
/data

0 directories, 0 files
(yjkim@yjkim-eks:N/A) [root@yjkim-eks-bastion-EC2 test]# ssh ec2-user@$N4 sudo cat /etc/fstab
#
UUID=0ccd3c9e-3e0f-4e59-ae3c-9498ef40c541     /           xfs    defaults,noatime  1   1
/dev/nvme1n1 /data xfs defaults,noatime 0 2

# 삭제 
eksctl delete nodegroup -c $CLUSTER_NAME -n ng2
```


## 삭제 

```sh
eksctl delete cluster --name $CLUSTER_NAME && aws cloudformation delete-stack --stack-name $CLUSTER_NAME
```