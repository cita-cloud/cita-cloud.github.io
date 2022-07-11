---
title: 在Rust中实现gRPC重试
date: 2022-07-10 12:53:28
categories: Rust
---

## 问题

`CITA-Cloud`采用了微服务架构，微服务之间以及对应用暴露的接口都采用了`gRPC`。

`gRPC`调用时可能会返回错误，需要对错误进行处理。经过讨论之后，我们采用的[方案](https://github.com/cita-cloud/rfcs/pull/7/files)是将错误码分成两层。应用层面的错误用单独的`status_code`来表示；`gRPC`本身的错误会用响应中的`Status`来表示。

前段时间碰到了可能是网络抖动造成的客户端返回`UNAVAILABLE`的现象。因为这个是`gRPC`本身的错误，应用层没办法处理，只能靠客户端重试来解决。

## gRPC重试

针对这个需求，可以使用`gRPC`本身就提供的拦截器功能。通过注入一个拦截器，检查每次调用的结果，如果返回错误（并不是每种错误都可以重试的，具体参见官方关于[错误码的描述](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)），则再次发起调用。

在`golang`这样的亲儿子上，甚至已经有[现成的库](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/master/retry)可以非常方便的做到这样的事情。

## gRPC重试 in Rust

因为`CITA-Cloud`主要使用`Rust`，结果搜索了一圈，震惊的发现竟然没有现成的库。在`Rust`这样造轮子情绪高涨的社区里，这是一个很不寻常的情况。

所幸一番搜索之后，发现了相应的[原因](https://github.com/hyperium/tonic/issues/733)，还是跟`Rust`的所有权特性有关系。

因为要在失败后重试，就要复制一份调用的请求参数。这个在其他语言里面根本不是个事，但是在`Rust`里就麻烦了。

`tonic`(一个纯`Rust`的`gRPC`实现)中一个接口的客户端函数原型为：
```
pub async fn store(
            &mut self,
            request: impl tonic::IntoRequest<super::Content>,
        ) -> Result<tonic::Response<super::super::common::StatusCode>, tonic::Status>
```

请求的类型是`tonic::IntoRequest<T>`（其中的`T`为请求中的应用层数据结构），这个类型是没有实现`Clone`的。

至于为什么不实现，开发者的解释是要考虑到`gRPC`的`stream`模式，`stream`中的请求是没法`Clone`的。

那非`stream`模式可以实现吗？答案也是不行，因为`gRPC`是基于`Http2`的，`Http2`总是`stream`的，因此单次调用模式其实就是只包含一个请求的`stream`。

## 解决方案

请求的类型`tonic::IntoRequest<T>`无法`Clone`，但是里面的`T`通常都是可以`Clone`的。

因此在`Rust`中像`golang`一样通过拦截器来非常优雅的实现重试是做不到了，但是用复杂一点的方法还是可以实现的。

其实说白了就是在应用层，按照最直接的方式来实现重试。在应用层多封装一层函数，其参数是应用层的请求类型`T`。调用接口之后，判断结果，如果是可重试的错误，则将类型`T`复制一份，重新发起调用。

当然这样实现的问题是重复的模式化的代码会非常多，所以具体实现还是用了一些技巧尽量让重复的代码少一点。

方案参考了[temporalio/sdk-core](https://github.com/temporalio/sdk-core/tree/master/client/src)，具体实现参见[代码](https://github.com/cita-cloud/cita_cloud_proto/pull/4/files)。

为了复用`retry`的逻辑，单独抽象出了`retry`模块。首先定义了`RetryClient`：
```
pub struct RetryClient<SG> {
    client: SG,
    retry_config: RetryConfig,
}
```
其中`client`是原始的`gRPC client`，`retry_config`是重试相关的选项，比如最多重试多少次等。

重试的逻辑在其成员方法`call_with_retry`中，里面主要用到了`FutureRetry`，即把整个调用封装成一个`Future`闭包，退避策略则使用了`ExponentialBackoff`。

当然最根本的还是前面提到的，要封装一层，使闭包的参数是可以`Clone`的。这部分都是一些模式化的代码，因此使用了一个宏来自动生成相关代码：
```
macro_rules! retry_call {
    ($myself:ident, $call_name:ident) => { retry_call!($myself, $call_name,) };
    ($myself:ident, $call_name:ident, $($args:expr),*) => {{
        let call_name_str = stringify!($call_name);
        let fact = || { async { $myself.get_client_clone().$call_name($($args,)*).await.map(|ret| ret.into_inner()) }};
        $myself.call_with_retry(fact, call_name_str).await
    }}
}
```

为了让`RetryClient`能够用于不同的`Service`，这里会把每个`Service`的客户端函数定义成一个`Trait`。比如：
```
#[async_trait::async_trait]
pub trait StorageClientTrait {
    async fn store(&self, content: storage::Content) -> Result<common::StatusCode, tonic::Status>;

    async fn load(&self, key: storage::ExtKey) -> Result<storage::Value, tonic::Status>;

    async fn delete(&self, key: storage::ExtKey) -> Result<common::StatusCode, tonic::Status>;
}
```
注意这里的函数原型是封装之后的。

然后为`RetryClient`相对应的特化类型实现这个`Trait`：
```
#[async_trait::async_trait]
impl StorageClientTrait for RetryClient<StorageServiceClient<InterceptedSvc>> {
    async fn store(&self, content: storage::Content) -> Result<common::StatusCode, tonic::Status> {
        retry_call!(self, store, content.clone())
    }

    async fn load(&self, key: storage::ExtKey) -> Result<storage::Value, tonic::Status> {
        retry_call!(self, load, key.clone())
    }

    async fn delete(&self, key: storage::ExtKey) -> Result<common::StatusCode, tonic::Status> {
        retry_call!(self, delete, key.clone())
    }
}
```
内容是完全使用前面的宏来实现的。第一个参数是`RetryClient`的`self`，第二个参数是`gRPC`接口的名称，后面是接口的参数。

这样就实现了一个尽量通用的`RetryClient`，然后以尽量少的重复代码来为多个`Service`都实现了重试的功能。

用法可以参见里面的测试代码。
```
let mock_client = TestClient::new(code);
let retry_client = RetryClient::new(mock_client, Default::default());
let result = retry_client.test(1).await;
```
首先按照原有的方法获取底层的`Client`；然后将其和`RetryConfig`一起放入`RetryClient`，得到带重试功能的客户端；用这个客户端调用前述`Trait`中封装的方法就会自带重试功能了。
