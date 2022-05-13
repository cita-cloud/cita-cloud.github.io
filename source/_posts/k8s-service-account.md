---
title: 在 Pod 中，访问 Kubernetes API 接口，并控制访问权限
date: 2022-05-13 12:06:40
categories:
---

## 问题描述

在实际的应用场景中，我们的应用程序运行在集群中（以 Pod 的形式存在），并且该应用程序将在集群中进行创建资源、修改资源、删除资源等等操作。

在 Pod 中，访问 Kubernetes API 的方法有很多，而通过 Client Libraries（程序类库）是官方推荐的做法，也是我们接下来将要学习的方法。

该笔记将记录：在 Pod 中，通过 Client Libraries 访问 Kubernetes API（管理 Kubernetes 集群）的方法，以及相关问题的解决办法。

## 解决方案

接下来我们将介绍例如，如果创建 ServiceAccount 对应用程序进行访问控制，只允许其查看 Pod 资源（即查看 Pod 列表和详细信息）

### 第 1 步、创建 ServiceAccount 资源

而这其中最大的问题是，如何进行合理的授权，即对于既定的用户或应用程序，如何允许或拒绝特定的操作？—— 通过 ServiceAccount 实现。

	# kubectl create serviceaccount myappsa


### 第 2 步、引用 ServiceAccount 资源

定义一个 Pod，使用为 myappsa 的 ServiceAccount 资源：
```
	kubectl apply -f - <<EOF
	kind: Pod
	apiVersion: v1
	metadata:
	  name: myapp
	spec:
	  serviceAccountName: myappsa
	  containers:
	  - name: main
	    image: bitnami/kubectl:latest
	    command:
	    - "sleep"
	    - "infinity"
	EOF
```

ServiceAccount 是种身份，而在 Pod 中引用 ServiceAccount 则是将这种身份赋予 Pod 实例。而接下来的任务是给 ServiceAccount 这种身份赋予各种权限 —— 具体的做法便是将各种角色（Role）绑定（RoleBinding）到这个身份（ServiceAccount）上。

### 第 3 步、创建 Role 资源

定义名为 podreader 的 Role 资源，并定义其能够进行的访问操作：
```
	kubectl apply -f - <<EOF
	kind: Role
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: podreader
	rules:
	  - apiGroups: [""]
	    resources: ["pods"]
	    verbs: ["get", "list"]
	EOF
	
	# 或，直接使用命令创建
	# kubectl create role podreader --verb=get --verb=list --resource=pods -n default
```

### 第 4 步、将 Role 与 ServiceAccount 绑定

```
	kubectl apply -f - <<EOF
	kind: RoleBinding
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: podreaderbinding
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: Role
	  name: podreader
	subjects:
	  - kind: ServiceAccount
	    name: myappsa
	    namespace: default
	EOF
	
	# 或，从从命令行直接创建：
	# kubectl create rolebinding podreaderbinding --role=default:podreader --serviceaccount=default:myappsa --namesepace default -n default
```

### 第 5 步、访问 Kubernetes API 测试

通过 Service Account 相关信息来访问资源：

```
	# 通过我们运行的 kubectl 容器访问
	kubectl get pods                                    # 这里是为了体现 kubectl 并未通过 kubeconfig 的信息来访问集群，
	                                                    # 而是通过 /var/run/secrets/kubernetes.io/serviceaccount 来访问集群
	                                                    # kubectl 手册介绍该命令是如何查找访问信息的；
	
	# 通过 curl API 访问
	APISERVER=https://kubernetes.default.svc
	SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
	NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
	TOKEN=$(cat ${SERVICEACCOUNT}/token)
	CACERT=${SERVICEACCOUNT}/ca.crt
	curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/namespaces/default/pods
```

### 第 6 步、通过客户端类库访问集群

以 Java 客户端为例：

如果需要在集群内部访问集群：
1）按照如上示例，定义 ServiceAccount 资源，
2）并参照 [InClusterClientExample.java](https://github.com/kubernetes-client/java/blob/master/examples/examples-release-15/src/main/java/io/kubernetes/client/examples/InClusterClientExample.java) 示例代码

如果需要在集群外部访问集群：
1）按照如上示例，在远端集群定义 ServiceAccount 资源，
2）并参照 [java/KubeConfigFileClientExample.java](https://github.com/kubernetes-client/java/blob/master/examples/examples-release-15/src/main/java/io/kubernetes/client/examples/KubeConfigFileClientExample.java%20) 示例代码

## 补充说明

### 访问集群级别的资源

上面是命名空间内的访问控制设置，因为上述使用的是 Role 和 RoleBinding 资源，而对于集群范围的访问控制应该使用 ClusterRole 和ClusterRoleBinding 命令。

替换 kind 为 ClusterRole 及 ClusterRoleBinding 即可，其他部分与 Role、RoleBinding 类似，这里不再赘述。

### 通过 ServiceAccount 生成 Kubeconfig 的方法

如果需要使用 kubeconfig 文件，可通过 ServiceAccount 资源来创建，参考 [How to create a kubectl config file for serviceaccount](https://stackoverflow.com/questions/47770676/how-to-create-a-kubectl-config-file-for-serviceaccount%20) 讨论。

简而言之，kubeconfig 的 user 部分为 token，而非 client-certificate-data 与 client-key-data 参数。

## 参考文献

[Accessing the Kubernetes API from a Pod | Kubernetes](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/%20)
[dashboard - How to bind roles with service accounts - Kubernetes - Stack Overflow](https://stackoverflow.com/questions/59353819/how-to-bind-roles-with-service-accounts-kubernetes%20)
[A Quick Intro to the Kubernetes Java Client | Baeldung](https://www.baeldung.com/kubernetes-java-client%20)

