---
title: 在 Kubernetes 中，免费 HTTPS 证书（cert-manager）
date: 2022-04-01 19:28:06
categories:
---

问题描述
----

我们使用 Certbot 工具向 Let's Encrypt 免费申请并自动续期证书。

在 Kubernetes Cluster 中，我们使用 cert-manager 组件来实现。

该笔记将记录：在 Kubernetes Cluster 1.22 中，部署 cert-manager 1.7 组件，以及相关问题解决办法。

解决方案
----

### 环境信息

集群版本：Kubernetes Cluster v1.22.3-aliyun.1
组件版本：cert-manager v1.7（Supported Kubernetes versions: 1.18-1.23）

### 补充说明

1）作为系列部署资源，cert-manager 运行在 Kubernetes Cluster 中，并利用 CRD 来配置 CA 并请求证书；
2）部署方式：我们使用官方文档中推荐的 cmctl 命令，不再使用原始的 YAML 清单文件；
3）在部署 cert-manager 组件之后，需要创建代表 CA 的 Issuer 或 ClusterIssuer 资源；
4）在集群中部署多个 cert-manager 实例会出现意外行为（以前 v1.3 文档提到过，该版本不清楚是否存在该限制）；

### 准备工作

安装命令：
```bash
OS=$(go env GOOS); ARCH=$(go env GOARCH); curl -sSL -o cmctl.tar.gz https://github.com/cert-manager/cert-manager/releases/download/v1.7.2/cmctl-$OS-$ARCH.tar.gz
tar xzf cmctl.tar.gz
sudo mv cmctl /usr/local/bin
```

### 第一步、部署组件
```bash
cmctl x install --dry-run
```

### 第二步、验证安装
```bash
# cmctl check api --wait=2m
The cert-manager API is ready

# kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-75cf8df6b6-t2xns              1/1     Running   0          2m37s
cert-manager-cainjector-857f5bd88c-5xzd7   1/1     Running   0          2m37s
cert-manager-webhook-5cd99556d6-s6jf5      1/1     Running   0          2m37s
```

### 第三步、签发测试

验证方法：通过创建自签名证书，并检查证书是否能够自动签发（参考 [Verifying the Installation](https://cert-manager.io/docs/installation/verify/ ) 文档，以获取具体细节）

我们有使用手工方式来验证（我们通过文档中提到的社区工具来验证，但是失败）：
```
# cat > test-resources.yaml  <<EOF
apiVersion: v1
kind: Namespace
metadata:
    name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
    name: test-selfsigned
    namespace: cert-manager-test
spec:
    selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
    name: selfsigned-cert
    namespace: cert-manager-test
spec:
    dnsNames:
    - example.com
    secretName: selfsigned-cert-tls
    issuerRef:
    name: test-selfsigned
EOF

# kubectl apply -f test-resources.yaml
...
Status:
    Conditions:
    Last Transition Time:  2022-04-01T09:51:50Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
    Not After:               2022-06-30T09:51:50Z
    Not Before:              2022-04-01T09:51:50Z
    Renewal Time:            2022-05-31T09:51:50Z
    Revision:                1
...

# kubectl delete -f test-resources.yaml
namespace "cert-manager-test" deleted
issuer.cert-manager.io "test-selfsigned" deleted
certificate.cert-manager.io "selfsigned-cert" deleted
```

至此，已完成 cert-manager 部署，接下来便是使用 cert-manager 来申请 Let's Encrypt 证书（结合我们的需求）。

通过 cert-manager 申请 Let's Encrypt 证书
-----------------------------------

### 问题描述

前面步骤演示如何部署 cert-manager 组件，并成功申请自签名证书，但这并非我们的实际应用场景。

我们希望通过 cert-manager 组件，在集群内完成 Let's Encrypt 证书申请和管理：
1）我们使用 阿里云 DNS，并通过 DNS01 完成域名所有权认证（部分集群在内网，无法使用 HTTP01 认证）；

### 解决方案

需要阅读如下文档，以了解相关内容：
[cert-manager/Configuration/ACME](https://cert-manager.io/docs/configuration/acme/)
---- [cert-manager/Configuration/ACME/DNS01](https://cert-manager.io/docs/configuration/acme/dns01/)
-------- [cert-manager/Configuration/ACME/DNS01/Webhook](https://cert-manager.io/docs/configuration/acme/dns01/webhook/)

我们这里使用 [DEVmachine-fr/cert-manager-alidns-webhook](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook ) 来完成证书申请。

安装 Webhook 部分：
```
# 网络原因，所以我们提前下载 helm chart 文件
wget https://github.com/DEVmachine-fr/cert-manager-alidns-webhook/releases/download/alidns-webhook-0.6.1/alidns-webhook-0.6.1.tgz

# 查看相关变量
helm show values ./alidns-webhook-0.6.1.tgz

# 大多数变量不需要修改，除了 groupName 参数
helm -n cert-manager install alidns-webhook ./alidns-webhook-0.6.1.tgz \
    --set groupName=your-company.example.com
```

创建 ClusterIssuer 资源：
```
# 完成 DNS 质询需要访问阿里云接口来修改 DNS 记录，所以需要使用 KEY 与 TOKEN 来认证
kubectl create secret generic                      \
    alidns-secrets                                 \
    --from-literal="access-token=your-token"       \
    --from-literal="secret-key=your-secret-key"

# 创建 ClusterIssuer 资源
kubectl apply -f <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
    name: letsencrypt
spec:
    acme:
    # 修改，邮箱地址
    email: contact@example.com
    # 修改，生产地址：https://acme-v02.api.letsencrypt.org/directory
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
        name: letsencrypt
    solvers:
    - dns01:
        webhook:
            # 修改，应用我们刚才创建的 Secret 资源
            config:
                accessTokenSecretRef:
                key: access-token
                name: alidns-secrets
                regionId: cn-beijing
                secretKeySecretRef:
                key: secret-key
                name: alidns-secrets
            # 修改，需要填写在安装时指定的 groupName 信息
            groupName: example.com
            solverName: alidns-solver
EOF
```

# 在 Ingress 中，使用 HTTPS 证书：
```
# https://cert-manager.io/docs/usage/ingress/
kubectl apply -f <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    annotations:
    # 修改，暗示要使用的 issuer 资源，由管理员提供
    cert-manager.io/cluster-issuer: nameOfClusterIssuer
    name: myIngress
    namespace: myIngress
spec:
    rules:
    ...
    tls:
    - hosts:
    # 修改，需要签发证书的域名
    - example.com
    # 修改，保存证书的 Secret 资源（cert-manger 负责创建）
    secretName: myingress-cert
EOF
```

参考文献
----

[GitHub - DEVmachine-fr/cert-manager-alidns-webhook](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook )
[Installation | cert-manager](https://cert-manager.io/docs/installation/ )
[cmctl | cert-manager](https://cert-manager.io/docs/installation/cmctl/ )
[Securing Ingress Resources | cert-manager](https://cert-manager.io/docs/usage/ingress/ )













