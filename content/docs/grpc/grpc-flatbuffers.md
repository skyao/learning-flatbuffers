---
title: "宣布Flatbuffers序列化库对gRPC的即开即用支持"
linkTitle: "Flatbuffers对gRPC的支持"
weight: 10
date: 2022-09-05
description: >
  2017年gRPC宣布Flatbuffers序列化库对gRPC的支持
---



> 翻译自grpc官方博客文档： [Announcing out-of-the-box support for gRPC in the Flatbuffers serialization library | gRPC](https://grpc.io/blog/grpc-flatbuffers/)，该文章发表于2017年。



最近发布的 Flatbuffers 1.7 版本为 gRPC 引入了真正的零拷贝（zero-copy）支持，开箱即用。

Flatbuffers 是一个序列化库，它允许你访问序列化的数据，而不需要先将其解压或分配任何额外的数据结构。它最初是为游戏和其他资源受限的应用而设计的，但现在发现了更普遍的用途，包括谷歌内部的团队和其他公司，如 Netflix 和 Facebook。

Flatbuffers 通过直接使用 gRPC 的零拷贝分片缓冲区（slice buffers）来实现最大的吞吐量，用于普通的用例。传入的 rpc 可以直接从 gRPC 的内部缓冲区进行处理，构建一个新的消息将直接写入这些缓冲区，没有中间步骤。

目前，FlatBuffers的 C++ 实现中完全支持这一点，更多的语言将陆续推出。在 Go 中也有一个实现，它不完全是零拷贝，但分配成本仍然很低（见下文）。

## 使用实例

让我们看一个如何工作的例子。

### 使用 Flatbuffers 作为一个IDL

从声明 RPC 服务的 `.fbs` 模式（类似于`.proto`，如果你熟悉 protocol buffers）开始。

```protobuf
table HelloReply {
  message:string;
}

table HelloRequest {
  name:string;
}

table ManyHellosRequest {
  name:string;
  num_greetings:int;
}

rpc_service Greeter {
  SayHello(HelloRequest):HelloReply;
  SayManyHellos(ManyHellosRequest):HelloReply (streaming: "server");
}
```

要从中生成C++代码，请运行： `flatc --cpp --grpc example.fbs`，与 protocol buffers 中的情况很相似。

### 生成的服务器实现

服务器的实现与 protocol buffers  非常相似，只是现在请求和响应消息的类型是 `flatbuffers::grpc::Message<HelloRequest> *`。与 protocol buffers  不同的是，这些类型代表了一棵 C++ 对象的树，在这里它们只是底层 gRPC slice 中扁平对象的句柄。你可以直接访问这些数据:

```c++
auto request = request_msg->GetRoot();
auto name = request->name()->str();
```

建立 response 同样简单:

```c++
auto msg_offset = mb_.CreateString("Hello, " + name);
auto hello_offset = CreateHelloReply(mb_, msg_offset);
mb_.Finish(hello_offset);
*response_msg = mb_.ReleaseMessage<HelloReply>();
```

客户端代码与  protocol buffers  产生的代码相同，除了 FlatBuffer 的访问和构造代码。

请看这里的完整例子。要编译它，你需要 gRPC。同一个 repo 有一个类似的 Go 的例子。





