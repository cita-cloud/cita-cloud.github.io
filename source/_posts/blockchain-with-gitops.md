---
title: 区块链与GitOps
date: 2022-05-11 09:36:02
categories: 区块链
---

# 背景

曾经有用户在论坛里反馈过区块链系统启动过程比较复杂。

首先区块系统是一个对等网络，而传统系统一般都是`Client/Server`或者`Master/Slave`形态。

所以在区块链设计中就有一个常见的模式，把非对等形态的软件变为对等形态。比较简单的做法就是所有节点都是`Server`，同时又是其他节点的`Client`，可以认为是把常见的网络库提供的`Server/Client`形态的功能转换为对等网络。

这个方案麻烦的是生成配置文件，以及后续增加删除节点时修改配置文件。

需要提供已知的所有节点的网络信息。遍历所有的节点，为每个节点生成一个相关配置。找到本节点的网络信息，确定本节点的监听端口，用于启动`Server`；使用除自己之外的其他节点的信息，用于本节点作为`Client`去连接其他节点。

相当于`N * (N - 1)`个`Client/Server`的配置，且需要所有节点信息才能生成对应的配置文件，所以需要集中统一生成。

同样，增加删除节点的时候，也涉及到所有配置文件的修改，需要集中修改，生成新的配置文件，然后下发到所有的节点。

其次它是一个去中心化的系统，实际生产部署的时候是有多个参与方的，每个参与方负责一个节点，相互之间的协调和配合工作量比较大。且需要考虑参与方的机密信息不能泄露，因此需要分成多个步骤，将机密信息生成和协作产生区块链配置分开，进一步增加了操作的复杂程度。

# 问题

1. 参与方之间交互比较多，需要传递信息也比较多。不管是通过发邮件，还是通过其他方式传递，管理都比较困难。
   比如，需要认为回信息的方式来确认对方已经收到，如果没有及时反馈，发送方重复发送，且信息与之前的不一致，如何处理？如果有人冒充参与方发送了假冒的信息，如何甄别？
2. 因为区块链系统多方协作的特点，上线运营之后可能还会持续有参与方加入和退出，导致节点的配置不断变化。
   有新增节点，有节点退出，节点的ip地址或者端口可能发生变动。如何应用这些变更？如何同步到其他节点？如何记录历史上的变更？手工操作进行配置升级容易出错，且消耗时间比较长。如果不能做到配置变更时，及时完成节点配置的更新部署，导致节点配置落后或者错误，可能会让整个区块链系统存在安全风险。比如一个参与方已经退出了，但是其他参与方没有及时更新这个信息，导致已经退出的参与方仍然能够访问系统的信息。

# GitOps

`Git`是一个开源的分布式版本控制系统，分布式相比集中式的最大区别是`Git`没有中央版本库，每一位开发者都可以通过克隆远程代码库，在本地机器上初始化一个完整的代码版本，开发者可以把代码的修改提交到本地代码库，也可以把本地的代码库同步到远程的代码库。

`GitOps`是一种持续交付的方式。它的核心思想是将应用系统的配置和部署以声明性的方式存放在`Git`版本库中。将`Git`作为交付流水线的核心，开发人员只需要将修改提到至`Git`，使用`Git`来加速和简化应用程序部署和运维任务。通过`GitOps`，当使用`Git`提交应用系统的配置更改时，自动化的交付流水线会将这些更改应用到实际系统中。

将`GitOps`方法应用在持续交付流水线上，有诸多优势和特点：
- 自动保证实际的应用系统和`Git`仓库中的配置是一致的。
- 更快的部署时间和恢复时间。
- 稳定且可重现的回滚。`Git`中保存有历史的配置信息，有问题可以随时切回历史版本。

# 解决方案

通过`Git`来管理系统配置，使用`GitOps`实现持续交付，在传统系统中已经非常流行。

但是区块链系统比传统系统更加适合这种配置管理方式。因为其配置的产生是一个协作的过程，更能发挥`Git`作为团队协作工具的特点；参与方本地拥有完整配置，但是只把部分非机密信息共享给其他参与方也符合`Git`在分布式上的特点。

链的配置涉及到多个参与方，以及相互之间的信息交互，并就此对链的配置进行相应的修改。所以这是一个多方协作的场景，使用`Git`管理链的配置可以很好的解决现有方案的问题。

1. `Git`可以设置权限，只允许参与方查看和修改存放链配置的仓库。
2. 通过`push`命令推送本地修改到远程，安全且有回馈。
3. 可以通过`PR`等功能对修改进行审查，一个或者多个参与方都确认之后才能合并。
4. 其他参与方可以通过`git pull`拉取最新的配置。
5. `Git`本身可以记录修改的历史，何时何人做了什么修改都有记录。
   
注意：
1. 因为配置里面有`ip`，端口等比较的敏感信息，可以通过建立私有仓库来避免信息泄露的风险。
2. 可以通过`Github`/`Gitlab`/`Gitee`等提供的图形界面和扩展功能更加方便的对配置修改进行管理。

### 区块链声明式的配置方式

要实施`GitOps`，核心的一点是要将区块链的配置方式改造为声明式。

如果配置方式仍然是命令式的，比如，增加一个节点，删除一个节点等。因为每个节点实施的顺序不同，可能会得出不同的结果，导致不同节点的配置不一致。

因此我们对区块链系统的配置项进行梳理：

![](configurations.png)

并据此制定一个声明式的配置数据结构：

![](struct.png)

配置变更时，不再下发配置变更动作，而是直接重新下发所有的配置信息，节点根据这个配置重新生成节点本地的配置文件。

### 链和节点Git仓库分离

理论上可以将一个区块链系统所有的配置信息都放在一个仓库中。

但是区块链去中心化的特性，链的配置需要公开，至少在参与方之间公开，以方便多个参与方都可以修改链的配置。但是节点配置中有很多机密信息，比如节点的私钥等，如果公开会造成安全方面的隐患。

因此，将链的配置信息和节点配置信息分开放在两个`Git`仓库中。

链的配置信息只包含可以公开的信息，比如账户地址等，可以放在一个公开的`Git`仓库中；而私钥等机密信息存在放节点配置中，放在参与方内部私有`Git`仓库中。

### 交付流水线

以增加一个节点为例。

- 新的参与方首先申请公开的存放链的配置信息的仓库的访问权限。
- 拉取最新的链的配置，并提交要增加的节点的公开信息，然后以`PR`的形式提交对链的配置的修改。
- 增加节点信息的PR经过审批之后，合并进最新的链的配置。
- 已有节点通过设置链的配置`Git`仓库的`Webhook`感知链的配置的变化。
- 通过本地的配置工具更新本地节点的配置文件，并提交至存放节点配置的`Git`仓库。
- 应用部署系统同样通过设置存放节点配置的`Git`仓库的`Webhook`感知节点配置的变化。
- 停掉已经存在的节点应用，并拉取最新的节点配置，重启节点应用，完成整个配置变更。

![](workflow.jpg)

# 演示

这里在`gitee`上创建[链级配置的仓库](https://gitee.com/cita-cloud/gitops-test-chain)。

配置工具为[cloud-config](https://github.com/cita-cloud/cloud-config)。

在公司内部的`Gitlab`上创建节点级配置仓库。

使用`Jekins`，并在相应的`Git`仓库设置`Webhook`来自动触发流水线。

区块链系统运行环境为`k8s`，并且集群中安装[ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/)，用于支持`GitOps`操作。

## 链级别配置

### 创建仓库

在`gitee`上创建仓库`gitops-test-chain`，保存链级配置。设置`master`分支不能直接`push`，只能`PR`的方式合入，并需要多个人的审批才能合入。

### 参与方加入

各个参与方申请该仓库的权限，并进行相关的设置，比如设置`SSH Key`。

### 链的发起方初始化链级配置

初始化链级配置并提交至`gitops-test-chain`。

```
$ cloud-config init-chain --chain-name gitops-test-chain
$ cloud-config init-chain-config --chain-name gitops-test-chain --consensus_tag v6.4.0 --controller_tag v6.4.0 --executor_tag v6.4.0 --kms_tag v6.4.0 --network_tag v6.4.0 --storage_tag v6.4.0
$ cd gitops-test-chain/
$ git init
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        .gitignore
        accounts/
        ca_cert/
        certs/
        chain_config.toml

nothing added to commit but untracked files present (use "git add" to track)
$ git add .
$ git commit -m "init chain config"
$ git remote add origin git@gitee.com:cita-cloud/gitops-test-chain.git
$ git push -u origin master
```

### 设置超级管理员

超级管理员拉取最新配置，生成自己的账号，并将地址设置为为链的`admin`。
```
// 拉取最新的链的配置
$ git clone git@gitee.com:cita-cloud/gitops-test-chain.git
// 切换到新的分支
$ cd gitops-test-chain
$ git checkout -b set-admin
$ cd ..

// 生成admin账户并设置到链级配置中
$ cloud-config new-account --chain-name gitops-test-chain
key_id:1, address:4dafebf719f2a0439387a231977e5209fabb0cca
$ cloud-config set-admin --chain-name gitops-test-chain --admin 4dafebf719f2a0439387a231977e5209fabb0cca

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "set admin"
$ git push --set-upstream origin set-admin
```

创建`PR`:

![](set-admin.png)

经过评审之后合并。

### 参与方1设置共识账户

参与方1拉取最新配置，生成自己的共识账号，并将地址添加为链的`validator`。

```
// 拉取最新的链的配置
$ git clone git@gitee.com:cita-cloud/gitops-test-chain.git

// 切换到新的分支
$ cd gitops-test-chain
$ git checkout -b add-validator1
$ cd ..

// 生成共识账户并设置到链级配置中
$ cloud-config new-account --chain-name gitops-test-chain
key_id:1, address:a746b04e30709203d3c8aadcca31bb024bbcf5df
$ cloud-config append-validator --chain-name gitops-test-chain --validator a746b04e30709203d3c8aadcca31bb024bbcf5df

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "append validator1"
$ git push --set-upstream origin add-validator1
```

创建`PR`:

![](append-validator.png)

经过评审之后合并。

### 参与方2设置共识账户

参与方2拉取最新配置，生成自己的共识账号，并将地址添加为链的`validator`。

操作同参与方1，这里不再赘述。

### 链的发起方关闭链级配置

所有共识参与方都已经添加过共识账户。

链的参与方将链级配置的`stage`设置为`finalize`。

此后将无法添加共识账户。

```
// 拉取最新的链的配置，并创建新的分支
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ git checkout -b set-stage-finalize
$ cd ..

// 将链级配置的stage设置为finalize
$ cloud-config set-stage --chain-name gitops-test-chain

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "set stage finalize"
$ git push --set-upstream origin set-stage-finalize
```

创建`PR`:

![](final.png)

经过评审之后合并。

### 参与方1添加节点网络信息

```
// 拉取最新的链的配置，并创建新的分支
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ git checkout -b set-node0
$ cd ..

// 添加节点0的网络信息
$ cloud-config append-node --chain-name gitops-test-chain --node www.node0.com:40000:node0:k8s

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "set node0"
$ git push --set-upstream origin set-node0
```

创建`PR`:

![](set-node.png)

经过评审之后合并。

### 参与方2添加节点网络信息

操作同参与方1，这里不再赘述。

### 超级管理员创建CA证书

因为网络采用`network_tls`，因此需要创建链的`CA`证书，并为每个节点创建证书。

```
// 拉取最新的链的配置，并创建新的分支
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ git checkout -b create-ca
$ cd ..

// 创建CA证书
$ cloud-config create-ca --chain-name gitops-test-chain

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "create ca"
$ git push --set-upstream origin create-ca
```

创建`PR`:

![](create-ca.png)

经过评审之后合并。

### 参与方1创建CSR

为了不暴露参与方证书的私钥信息，这里采用`Certificate Signing Request`的方式申请证书。

```
// 拉取最新的链的配置，并创建新的分支
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ git checkout -b create-csr-node0
$ cd ..

// 创建CA证书
$ cloud-config create-csr --chain-name gitops-test-chain --domain node0

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "create csr for node0"
$ git push --set-upstream origin create-csr-node0
```

创建`PR`:

![](create-csr.png)

评审后合并。

### 参与方2创建CSR

操作同参与方1，这里不再赘述。

### 超级管理员处理CSR

```
// 拉取最新的链的配置，并创建新的分支
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ git checkout -b sign-csr
$ cd ..

// 处理CSR，签名
$ cloud-config sign-csr --chain-name gitops-test-chain --domain node0
$ cloud-config sign-csr --chain-name gitops-test-chain --domain node1

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "sign csr"
$ git push --set-upstream origin sign-csr
```

创建`PR`:

![](sign-csr.png)

评审后合并。

## 节点配置

### 参与方1初始化节点

在内部的`GitLab`上创建节点配置仓库`gitops-test-chain-node0`。

创建`read_write_access token`，记录`token`和同时创建的`bot`账户，方便后续在流水线中拉取和提交代码。

```
// 拉取最新的链的配置
$ cd gitops-test-chain
$ git checkout master
$ git pull
$ cd ..

// 初始化node0节点配置
$ cloud-config init-node --chain-name gitops-test-chain --domain node0 --account a746b04e30709203d3c8aadcca31bb024bbcf5df
// 生成node0节点配置文件
$ cloud-config update-node --chain-name gitops-test-chain --domain node0
// 生成node0资源清单
$ cloud-config update-yaml --chain-name gitops-test-chain --domain node0 --storage-class nfs-client

// 提交节点初始化配置
$ cd gitops-test-chain-node0
$ git init
$ git add .
$ git commit -m "init node config"
$ git remote add origin https://project_xxx_bot:xxxxxxxx@git.XXX.com/xxx/gitops-test-chain-node0.git
$ git push -u origin master
```

### 参与方1设置流水线

在`Jekins`中创建新的流水线，设置源码为链级配置的仓库: `https://gitee.com/cita-cloud/gitops-test-chain.git`的`master`分支，并设置检出到子目录`gitops-test-chain`。

设置`WebHook`的`token`并设置过滤条件，只在`master`分支有更新的时候才触发流水线。

执行脚本类似前面的初始化节点配置：

```
set +e
rm -rf gitops-test-chain-node0
git clone https://project_xxx_bot:xxxxxxxx@git.XXX.com/xxx/gitops-test-chain-node0.git
docker run -i --rm -v `pwd`:`pwd` -w `pwd` citacloud/cloud-config:v6.4.0 cloud-config init-node --chain-name gitops-test-chain --domain node0 --account 6b9ac59d83e9d0744f6231d453e2b883b1819358
docker run -i --rm -v `pwd`:`pwd` -w `pwd` citacloud/cloud-config:v6.4.0 cloud-config update-node --chain-name gitops-test-chain --domain node0
docker run -i --rm -v `pwd`:`pwd` -w `pwd` citacloud/cloud-config:v6.4.0 cloud-config update-yaml --chain-name gitops-test-chain --domain node0 --storage-class nfs-client
cd gitops-test-chain-node0
git add .
git commit -m "update node0 config"
git push -u origin master
```

在链级配置的仓库中设置`WebHook`:

![](webhook.png)

### 参与方2初始化节点并设置流水线

操作同参与方1，这里不再赘述。

## 集群配置

节点配置仓库到`k8s`集群的同步使用`ArgoCD`。

安装和使用方法参见[文档](https://argo-cd.readthedocs.io/en/stable/getting_started/)。

创建节点应用，并设置自动同步配置：

```
argocd app create gitops-test-chain-node0 --repo https://project_xxx_bot:xxxxxxxx@git.XXX.com/xxx/gitops-test-chain-node0.git --path yamls --dest-server https://xxx.xxx.xxx.xxx:6443 --dest-namespace default --sync-policy auto

argocd app create gitops-test-chain-node1 --repo https://project_yyy_bot:yyyyyyyy@git.YYY.com/yyy/gitops-test-chain-node1.git --path yamls --dest-server https://yyy.yyy.yyy.yyy:6443 --dest-namespace default --sync-policy auto
```

## 测试

### 参与方3添加节点网络信息

```
// 拉取最新的链的配置，并创建新的分支
$ git clone git@gitee.com:cita-cloud/gitops-test-chain.git
$ cd gitops-test-chain
$ git checkout -b set-node2
$ cd ..

// 添加节点0的网络信息
$ cloud-config append-node --chain-name gitops-test-chain --node www.node2.com:60000:node2:k8s

// 提交修改
$ cd gitops-test-chain
$ git add .
$ git commit -m "set node2"
$ git push --set-upstream origin set-node2
```

创建`PR`:

![](add-node2.png)

审核后合并。

此后`node0`和`node1`会经由`WebHook`得知链的配置发生变更，并通过`Jenkins`流水线自动更新节点配置文件，再通过`Argocd`，自动将新配置应用到集群中。

因为我们并没有真正的运行`node2`，所以`node0`和`node1`的`network`微服务应该会报连接错误，如下：
```
 2022-04-25T12:59:31.881922Z  INFO network::peer: connecting.. peer=gitops-test-chain-node2 host=gitops-test-chain-node2-nodeport port=40000                                                                │
│ 2022-04-25T12:59:36.885651Z  INFO network::peer: connecting.. peer=gitops-test-chain-node2 host=gitops-test-chain-node2-nodeport port=40000
```                                                          
