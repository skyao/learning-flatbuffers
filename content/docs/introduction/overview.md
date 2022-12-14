---
title: "FlatBuffers概述"
linkTitle: "概述"
weight: 10
date: 2022-09-05
description: >
  FlatBuffers概述：为什么使用FlatBuffers而不是Protocol Buffers或者json
---



> 内容援引： https://google.github.io/flatbuffers/



FlatBuffers 是一个高效的跨平台序列化库，适用于C++、C#、C、Go、Java、Kotlin、JavaScript、Lobster、Lua、TypeScript、PHP、Python、Rust 和 Swift。它最初是在谷歌创建的，用于游戏开发和其他性能关键型应用。

它在 GitHub 上以 Apache 许可证 v2 的形式提供开源（见LICENSE.txt）。

### 为什么使用FlatBuffers

- **无需解析/解包即可访问序列化数据**：FlatBuffers 与众不同之处在于，它在一个扁平（flat）的二进制缓冲区中描述分层数据，在这种情况下数据仍然可以直接访问，而无需解析（parsing）/解包（unpacking），同时也仍然支持数据结构的演变（向前/向后兼容）。

- **内存效率和速度** - 访问你的数据所需的唯一内存是缓冲区的内存。它需要0个额外的分配（在C++中，其他语言可能有所不同）。FlatBuffers 也很适合与 mmap（或流媒体）一起使用，只需要将缓冲区的一部分放在内存中。访问速度接近于原始结构体的访问速度，只有一个额外的中介（一种vtable）以支持格式演进和可选字段。它的目标是那些不想花费时间和空间（许多内存分配）来访问或构建序列化数据的项目，例如游戏或任何其他对性能敏感的应用程序。详见基准。

- **灵活**：可选字段意味着你不仅可以得到很好的向前和向后的兼容性（对于长期存在的游戏越来越重要：不必在每个新版本中更新所有数据！）。它还意味着你在写什么数据和不写什么数据，以及如何设计数据结构方面有很多选择。

- **微小的代码足迹**：生成的代码量小，而且只有一个很小的头文件作为最小的依赖，这非常容易集成。同样，详情请见基准部分。

- **强类型化**：错误发生在编译时，而不是手动编写重复的、容易出错的运行时检查。可以为你生成有用的代码。

- **使用方便**：生成的C++代码允许简洁的访问和构造代码。然后还有可选的功能，如果需要的话，可以在运行时有效地解析模式和类似 JSON 的文本表示法（比其他 JSON 解析器更快，更节省内存）。

  Java、Kotlin和Go代码支持对象重用。C#有高效的基于结构体的访问器。

- **无依赖的跨平台代码** ：C++代码可以在任何最新的gcc/clang和VS2010下运行。附带测试和样本的构建文件（Android .mk文件，以及所有其他平台的cmake）。

### 为什么不使用协议缓冲区，或者 ... ?

Protocol Buffers  确实与 FlatBuffers 比较相似，主要区别在于 FlatBuffers 不需要在访问数据之前进行解析/解包步骤，而是采用二级表示法，通常再加上每对象分配内存。代码也大了一个数量级。Protocol Buffers 没有可选的文本导入/导出。

### 但所有的酷孩子都在使用JSON!

JSON是非常可读的（这就是为什么我们使用它作为我们的可选文本格式），并且在与动态类型的语言（如JavaScript）一起使用时非常方便。然而，当从静态类型语言序列化数据时，JSON不仅有运行时效率低下的明显缺点，而且由于其动态类型的序列化系统，迫使你写更多的代码来访问数据（反直觉）。在这种情况下，它只对那些事先很少甚至没有关于需要存储什么数据的信息的系统是一个更好的选择。

如果你确实需要存储不符合模式的数据，FlatBuffers也提供了一个无模式（自我描述）的版本!

在 白皮书 中阅读更多关于 FlatBuffers 的"why"。

### 谁使用FlatBuffers？

- Cocos2d-x，第一大开源移动游戏引擎，用它来序列化他们所有的游戏数据。
- Facebook 在他们的安卓应用中使用它进行客户-服务器通信。他们有一篇很好的文章解释了它是如何加速加载他们的帖子的。
- Google 的 Fun Propulsion Labs 在他们所有的库和游戏中广泛地使用它。

### 使用方法简述

本节是对如何使用本系统的快速概述。后面的章节提供了更深入的使用指南。

- 编写模式文件，允许你定义可能想要序列化的数据结构。字段可以有一个标量（scalar）类型（各种大小的 ints/floats），或者它们可以是：字符串；任何类型的数组；对另一个对象的引用；或者，一组可能的对象（unions）。字段是可选的，并且有默认值，所以它们不需要出现在每个对象实例中。

- 使用 `flatc`（FlatBuffer 编译器）生成一个 C++头（或 Java/Kotlin/C#/Go/Python......类），用 helper 类来访问和构造序列化的数据。这个头文件（例如 `mydata_generated.h`）只依赖于定义了核心功能的 `flatbuffers.h`。

- 使用FlatBufferBuilder类来构造一个平面二进制缓冲区。生成的函数允许你向这个缓冲区递归地添加对象，通常就像调用一个函数一样简单。

- 存储或发送你的缓冲区的某个地方!

- 当读回它时，你可以从二进制缓冲区获得根对象的指针，并从那里用 `object->field()` 方便地原地遍历它。





