---
title: 在 Kubernetes 中，部署云原生 NFS 存储
date: 2022-03-14 11:25:55
categories: 云计算
---

# 问题描述

我们常用的存储软件（比如 NFS、Ceph、EdgeFS、YugabyteDB 等等）并不具备（或仅具备部分）高可用、自愈、自动扩展等等特性。

# 解决方案

Rook，是开源的云原生存储编排器，提供平台，框架，支持“多种本机存储解决方案”与“云原生环境”集成。Rook 基于底层云原生平台对这些存储软件进行强化。通过使用“__底层的云原生容器管理、调度、协调平台提供的__”基础设施，来为存储服务添加 自我管理、自我缩放、自愈的 等等特性。它通过自动部署、引导、配置、部署、缩放、升级、迁移、灾难恢复，监控，资源管理来实现。

该笔记将记录在 Kubernetes 中如何部署 Rook 服务，底层使用 NFS 存储，以及常见问题解决办法。

## 环境要求

1. Kubernetes v1.16 or higher
1. The desired volume to export needs to be attached to the NFS * server pod via a PVC
1. NFS client packages must be installed on all nodes where * Kubernetes might run pods with NFS mounted. 

## 环境信息

1. Rook NFS v1.7（03/14/2022），建议阅读 [NFS Docs/v1.7](https://rook.io/docs/nfs/v1.7/%20) 文档以了解更多细节，这里我们仅记录适用于我们环境的部署过程。
1. Kubernetes HA Cluster 1.18.20, worker k8s-storage as dedicated storage node

## 关于存储
1. 简单的拓扑结构为 Normal Pod ⇒ Storage Class ⇒ NFS Server ⇒ PVC ⇒ PV (hostPath) 所以我们以 hostPath 方式来提供最终的存储；
1. 通过专用的存储节点，即 Kubernetes Worker 但是不会向该节点调度 Pod 实例（通过 Taint 及 Namespace defaultTolerations 来实现）；

## 准备工作 

```bash
# Taint node，以专用于存储
kubectl taint nodes k8s-storage dedicated=storage:NoSchedule

# 开启 PodNodeSelector,PodTolerationRestriction 插件（不再细述）
kube-apiserver ... --enable-admission-plugins=NodeRestriction,PodNodeSelector,PodTolerationRestriction ...
```

## 第一步、部署 NFS Operator 组件

```bash
git clone --single-branch --branch v1.7.3 https://github.com/rook/nfs.git
cd nfs/cluster/examples/kubernetes/nfs
kubectl create -f crds.yaml
kubectl create -f operator.yaml

# kubectl get pods -n rook-nfs-system 
NAME                                 READY   STATUS    RESTARTS   AGE
rook-nfs-operator-794b5c98bd-rc8lv   1/1     Running   0          8m31s
```

补充说明：
* Operator 是否调度到 k8s-storage（专用存储节点）并不重要；

## 第二步、创建 NFS Server 服务

rbac.yaml
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name:  rook-nfs
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: kubernetes.io/hostname=k8s-storage
    scheduler.alpha.kubernetes.io/defaultTolerations: '[{"operator": "Exists", "effect": "NoSchedule", "key": "dedicated"}]'
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rook-nfs-server
  namespace: rook-nfs
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rook-nfs-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get"]
  - apiGroups: ["policy"]
    resources: ["podsecuritypolicies"]
    resourceNames: ["rook-nfs-policy"]
    verbs: ["use"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups:
    - nfs.rook.io
    resources:
    - "*"
    verbs:
    - "*"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rook-nfs-provisioner-runner
subjects:
  - kind: ServiceAccount
    name: rook-nfs-server
     # replace with namespace where provisioner is deployed
    namespace: rook-nfs
roleRef:
  kind: ClusterRole
  name: rook-nfs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

nfs-server.yaml
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: rook-nfs-pv
  namespace: rook-nfs
  labels:
    type: local
spec:
  storageClassName: manual
  claimRef:
    name: rook-nfs-pvc
    namespace: rook-nfs
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/srv/k8s-storage-nfs-rook-pv"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-storage
---
# A default storageclass must be present
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rook-nfs-pvc
  namespace: rook-nfs
spec:
  volumeName: "rook-nfs-pv"
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
---
apiVersion: nfs.rook.io/v1alpha1
kind: NFSServer
metadata:
  name: rook-nfs
  namespace: rook-nfs
spec:
  replicas: 1
  exports:
  - name: share-01
    server:
      accessMode: ReadWrite
      squash: "none"
    persistentVolumeClaim:
      claimName: rook-nfs-pvc
  annotations:
    rook: nfs
```

查看结果：
```
# kubectl -n rook-nfs get nfsservers.nfs.rook.io
NAME       AGE   STATE
rook-nfs   2m    Running

# kubectl -n rook-nfs get pod -l app=rook-nfs -o wide
NAME         READY   STATUS    RESTARTS   AGE    IP               NODE      NOMINATED NODE   READINESS GATES
rook-nfs-0   2/2     Running   0          6m8s   192.168.59.130   k8s-w03   <none>           <none>
```

## 第三步、使用 NFS 存储

storage-class.yaml
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    app: rook-nfs
  name: rook-nfs-share-01
parameters:
  exportName: share-01
  nfsServerName: rook-nfs
  nfsServerNamespace: rook-nfs
provisioner: nfs.rook.io/rook-nfs-provisioner
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

testing.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rook-nfs-pv-claim
spec:
  storageClassName: "rook-nfs-share-01"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
---
# cat nfs/cluster/examples/kubernetes/nfs/busybox-rc.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nfs-demo
    role: busybox
  name: nfs-busybox
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nfs-demo
      role: busybox
  template:
    metadata:
      labels:
        app: nfs-demo
        role: busybox
    spec:
      containers:
        - image: busybox
          command:
            - sh
            - -c
            - "while true; do date > /mnt/index.html; hostname >> /mnt/index.html; sleep $(($RANDOM % 5 + 5)); done"
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            # name must match the volume name below
            - name: rook-nfs-vol
              mountPath: "/mnt"
      volumes:
        - name: rook-nfs-vol
          persistentVolumeClaim:
            claimName: rook-nfs-pv-claim
```

# 补充说明

Pod 通过 Service 进行 NFS 挂载：
```bash
# kubectl -n rook-nfs get service rook-nfs 
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)            AGE
rook-nfs   ClusterIP   10.111.156.207   <none>        2049/TCP,111/TCP   135m
```

# 调试追踪

```
kubectl -n rook-nfs-system logs -l app=rook-nfs-operator

# NFS Server
kubectl -n rook-nfs logs rook-nfs-0 nfs-server                                

# Storage Class
kubectl -n rook-nfs logs rook-nfs-0 nfs-provisioner                           
```
# 参考文献

* [Default Toleration at Namespace Level | by Zhimin Wen | Medium](https://zhimin-wen.medium.com/default-toleration-at-namespace-level-f66dd3da4451 )
* [DaemonSet not respecting Namespace defaultTolerations · Issue #94722 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/issues/94722 )
* [Taints and Tolerations | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/ )
* [plugins - k3s node restriction for namespace - Stack Overflow](https://stackoverflow.com/questions/69449258/k3s-node-restriction-for-namespace )
* [Rook NFS/v1.7.3/Network Filesystem (NFS) Quickstart](https://rook.io/docs/nfs/v1.7/quickstart.html)
