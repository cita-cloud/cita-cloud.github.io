---
title: 技术沙龙第三期
date: 2021-11-14
categories: 智能合约
---
U2：

今天分享下openzeppelin的Upgradeable Smart Contract，就是可升级智能合约

U2：

我们先看下如何用他这个提供的插件来写智能合约

U2：

然后再来介绍下里面的原理和实现逻辑

U2：

首先我看一个正常的合约

U2：

比如一个合约里有他的初始化构造方法，我们要把这个构造方法，替换成initialize方法

U2：

我们先不考虑为什么需要这么做，我们先看下他的使用，后面我们再去看他的原理

U2：


![](contract.png)

比如这个MYcontract合约，我们就把他的构造方法换成initialize

U2：

然后能因为构造方法其实只能被调用一次

U2：

![](contract1.png)

为了防止出现被错误调用，所以这里加了一些条件限制

U2：

我记得很早以前某篇文章里介绍智能这种编程方式也叫，面向条件的编程。

U2：

大家感兴趣可以去查查这个面向条件编程，因为无论是普通合约逻辑的编写，还是在安全方面做的一些保护，比如防重入攻击，都有点这个条件编程的思想。

U2：

我们接着介绍

U2：

其实每次这样去写这个initialize的方法就很麻烦

U2：

![](contract2.png)

所以这个openz就帮我们实现了一些逻辑，我们只要继承这个就好了

U2：

那咱们平时写合约的时候，实际上还有一些合约继承的东西

U2：

这种情况能，就要稍微处理下

![](contract3.png)

U2：

然后如果咱们用了erc20这种合约咋办呢？

U2：

这个其实openz也提供了他标准的可升级的erc20的实现，我们把原来继承erc20，直接替换掉就好了

![](interface.png)

U2：

然后能还要注意一点，就是在变量声明的时候，不要做初始化，Avoiding Initial Values in Field Declarations

U2：

比如这个

![](contract4.png)
U2：

这种其实就相当于，我们写了一个构造方法，然后在构造方法里初始化这个storage

U2：

![](contract5.png)
就需要把他改成上面这样的

U2：

还有一种是常量

![](contract6.png)
U2：

这种不一样的，他实际上是编码在合约ｃｏｄｅ里的

U2：

我们在部署合约的时候，evm实际上在完成构造方法后，构造方法这段代码实际上就没有保存在合约地址下的code的，因为反正只是在部署的时候用。

U2：

但是初始化的常量实际上会跟随这个合约code，保存在合约地址下的代码里。

U2：

然后这个初始化里构造的变量呢，也是以storage的形式存下来的。

U2：

然后能，在初始化合约的时候，可能在初始化方法中会创建一个新的合约实例

U2：

![](contract7.png)

比如这种

U2：

这能里面的这个erc20合约实际上是不支持升级的

U2：

如果想要这个合约支持升级怎么处理呢？

U2：

实际就是先部署里面这个合约，然后在初始化的时候把这个里面的合约地址再传进去就好了

![](contract8.png)

U2：

然后能，咱们在升级合约的时候也要注意几点

U2：

第一不要更改原有的storage

U2：

比如原来是这样

U2：

```
contract MyContract {
    uint256 private x;
    string private y;
}
```

U2：

我们不能把这个变量类型改掉

U2：

```
contract MyContract {
    string private x;
    string private y;
}
```

U2：

也不能换顺序

U2：

```
contract MyContract {
    string private y;
    uint256 private x;
}
```

U2：

也不能在前面插入一个变量

U2：

```
contract MyContract {
    bytes private a;
    uint256 private x;
    string private y;
}
```

U2：

能做的是，可以给这个变量换个名字，或者在后面追加一个变量

U2：

比如这个

U2：

```
contract MyContract {
    uint256 private x;
    string private y;
    bytes private z;
}
```

U2：

就是加在最后

U2：

当然还有其他的错误使用情况，大家可以看他们的官方文档

U2：

然后能这个原理是怎么实现的呢

U2：

其实很简单

U2：

他这个就是用了一个代理合约

U2：

```
User ---- tx ---> Proxy ----------> Implementation_v0
|
------------> Implementation_v1
|
------------> Implementation_v2
```

U2：

这个我们用户调用的一直都是这个代理合约

U2：

然后具体的实现合约可以换掉，然后把地址绑定到这个代理合约之类

U2：

我们看下这个代理合约的大致的逻辑思路

U2：

```
assembly {
    let ptr := mload(0x40)

    // (1) copy incoming call data
    calldatacopy(ptr, 0, calldatasize)

    // (2) forward call to logic contract
    let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
    let size := returndatasize

    // (3) retrieve return data
    returndatacopy(ptr, 0, size)

    // (4) forward return data back to caller
    switch result
    case 0 { revert(ptr, size) }
    default { return(ptr, size) }
}
```

U2：

这个是assembly写的

U2：

这个第一步就是把calldata拷贝出来，就是交易里的data，第二步骤就是用这个delegatecall，然后第三就是获取return data，第四就是再吧这个returndata 返回

U2：

我们主要看这个delegatecall

U2：

solidity里有几种call，大家可以对比下这几个的区别

U2：

我们主要看这个delegatecall

U2：

这个能，其实就是把目标合约的code在当前合约的环境下执行，使用当前合约的storage

U2：

就是说逻辑合约的代码其实是被执行了，但是逻辑合约的storage其实是没有用的

U2：

他用的是在代理合约的storage

U2：

所以能，其实合约在初始化的时候，如果用solidity的构造方法，这个storage就留在逻辑合约里了，所以呢前面的把构造合约该成init就是这个道理，通过代理合约来调用iinit

U2：

然后这些storage就留在这个代理合约里了

U2：

然后后面就可以持续升级

U2：

刚才讲到在代理合约里保存逻辑合约的stroage

U2：

这里就有一个问题，就是原来代理合约因为也要保存逻辑合约的地址

U2：

比如在代理合约里声明了一个stroage

U2：

然后因为在solidity实现的时候，他会把这个storage给一个postion，然后实际上是按照他声明的位置来确定最终在合约下面的抽象模型里的kv里存的位置的

U2：

比如我们声明了三个变量

U2：

```
address a;
address b;
address c;
```

U2：

这三个实际上存在哪里是跟他的position有关的，就是变量的顺序

U2：

最终保存的时候我记得好像是按照合约地址＋position来作为这个storage的key的，然后再用mpt树做处理，具体这的细节，大家可以以后去研究。

U2：

然后这里可能就有一个问题

![](table.png)

U2：

就是proxy这里的storage，比如第一个和逻辑合约里的第一个storage，因为都是第一个，都存到proxy合约里

U2：

所以这里就冲突了

U2：

怎么办呢？

U2：

![](table1.png)

我们给他一个随即的slot就可以了

U2：

这里可以参考https://eips.ethereum.org/EIPS/eip-1967

U2：

原理很简单，实际就是算一个随机的数，然后用solidity的assembly语言里的一个设置stroage的命令，设置这个值就好了

U2：

这样就解决了代理合约和实现合约里的stroage冲突的问题

U2：

然后在实现合约里，因为stroage实际无论怎么升级都是公用的代理合约里的

U2：

所以这里就不能修改原来的stroage，要不然就冲突了

![](table2.png)
U2：

然后在使用上，openz和truffle也好，还有和一些其他的合约开发工具都是做了集成，使用起来都还是比较方便的

U2：

我的介绍就到这里

U2：

谢谢大家