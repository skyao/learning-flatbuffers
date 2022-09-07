---
title: "FlatBuffers内幕"
linkTitle: "内幕"
weight: 20
date: 2022-09-05
description: >
  FlatBuffers内幕
---



> https://google.github.io/flatbuffers/flatbuffers_internals.html



本节对于FlatBuffers的使用是完全可选的。在正常使用中，你不应该需要这里的信息。然而，如果你有兴趣，它应该让你更加了解为什么FlatBuffers既高效又方便。



## 格式组件

FlatBuffer 是一种二进制文件和内存格式，主要由不同大小的标量组成，都是按照自己的大小对齐。每个标量也总是以小端（little-endian ）格式表示，因为这与今天所有常用的CPU相对应。FlatBuffers 也可以在大端（big-endian）机器上工作，但由于有额外的字节交换本征，速度会稍慢。

假设满足以下条件，以确保跨平台的互操作性。

- 浮点数使用二进制的 `IEEE-754` 格式。

- 有符号的整数使用二进制补码表示。

- 浮点数的编码与整数的编码相同。

该格式故意留下了很多关于事物在内存中具体位置的细节，例如，表中的字段可以有任何顺序，而对象在某种程度上可以以多种顺序存储。这是因为格式不需要这些信息来提高效率，它为优化和扩展留下了空间（例如，字段可以以一种最紧凑的方式打包）。相反，该格式只用偏移和相邻关系来定义。这可能意味着在相同的输入值下，两个不同的实现可能会产生不同的二进制文件，而这是完全有效的。

## 格式识别

格式也不包含用于格式识别和版本管理的信息，这也是设计上的问题。FlatBuffers 是一个静态类型的系统，这意味着缓冲区的用户需要知道它是什么类型的缓冲区。当然，FlatBuffers 可以在需要时被包裹在其他容器内，或者你可以使用它的 union 功能来动态地识别存储的多个可能的子对象。此外，如果需要完整的反射能力，它可以和模式分析器一起使用。

版本管理是格式的内在组成部分（字段的可选性/可扩展性），所以格式本身不需要版本号（在某种意义上，它是一种元格式）。我们希望这个格式可以容纳所有需要的数据。如果有必要进行格式上的突破性改变，它将成为一种新的格式，而不仅仅是一种变化。

## 偏移量

最重要的和通用的偏移量 offset 类型（见 `flatbuffers.h`）是 `uoffset_t`，目前它总是一个 `uint32_t`，用来指代所有的tables / unions / strings / vectors（这些从来不是内连/ in-line 存储的）。32位是故意的，因为我们想保持格式在32位和64位系统之间的二进制兼容，而64位的偏移量会使几乎所有的使用都变得庞大。这个格式的一个版本带有64位（或16位）的偏移量，在需要时很容易设置。无符号意味着它们只能指向一个方向，通常是向前（指向一个更高的内存位置）。任何向后的偏移都会被明确地标记为这样。

格式以一个 `uoffset_t` 开始，指向缓冲区的根表。

我们有两种对象，结构体(struct)和表(table)。

## 结构体

这是最简单的，如前所述，用于简单的数据，受益于额外的效率，不需要版本/可扩展性。它们总是内联存储在其父类（结构体、表或向量）中，以获得最大的紧凑性。结构体定义了一个一致的内存布局，其中所有组件都按照其大小对齐，而结构体则按照其最大的标量成员对齐。这与底层编译器的对齐规则无关，以保证跨平台兼容的布局。然后，这种布局在生成的代码中被强制执行。

## 表

与结构体不同的是，这些表不是以内联方式存储在其父体中，而是通过偏移量来引用。

它们以一个 `soffset_t `开始，指向一个 vtable。这是 `uoffset_t` 的一个有符号版本，因为 vtables 可以存储在相对于对象的任何地方。这个偏移量是从对象的起始点上减去（而不是加上）vtable 的起始点。这个偏移量后面是作为对齐标量（或偏移量）的所有字段。与结构体不同，并非所有字段都需要出现。没有固定的顺序和布局。如果用户明确地将相同的偏移量序列化两次，一个表可能包含指向相同值的字段偏移量。

为了能够不受这些不确定性的影响而访问字段，我们通过一个偏移量的 vtable。Vtables 在任何碰巧拥有相同 Vtables 值的对象之间共享。

 vtable 的元素都是 `voffset_t` 类型的，它是一个 `uint16_t`。第一个元素是 `vtable` 的大小，单位是字节，包括 `size` 元素。第二个是对象的大小，以字节为单位（包括 vtable 的偏移）。这个大小可以用于流媒体，知道要读多少个字节才能访问对象的所有内联字段。剩下的元素是 N 个偏移量，其中 N 是构建这个缓冲区的代码被编译时在模式中声明的字段数量（因此，表的大小是N+2）。

在生成的表的代码中，所有的访问函数都包含进入这个表的偏移量，作为一个常数。这个偏移量与第一个字段（元素的数量）进行核对，以防止较新的代码读取较旧的数据。如果这个偏移量超出了范围，或者 vtable 条目是0，这意味着这个对象中不存在这个字段，默认值被返回。否则，该条目被用作要读取的字段的偏移量。

## unit

Unit 被编码为两个字段的组合：一个代表 unit 选择的枚举和实际元素的偏移。FlatBuffers 保留了枚举常数 NONE（编码为0），表示没有设置 unit 字段。

## 字符串和vector

字符串是简单的字节vector，并且总是以null结束。向量存储为连续对齐的标量元素，前缀为32位元素计数（不包括任何 null 结尾）。两者都不在其父本中内联存储，而是通过偏移量来引用。如果用户明确地将同一个偏移量序列化两次，一个向量可以由多个偏移量组成，指向同一个值。

## 构建

目前的实现是逆向构建这些缓冲区（从缓冲区的最高内存地址开始），因为这大大减少了记账（bookkeeping）的数量，并简化了构建API。

## 代码示例

下面是一个为 `samples/monster.fbs` 生成的代码例子。下面是整个文件的内容，由注释分割开来。

```protobuf
// automatically generated, do not modify

#include "flatbuffers/flatbuffers.h"

namespace MyGame {
namespace Sample {
```

支持嵌套命名空间。

```protobuf
enum {
  Color_Red = 0,
  Color_Green = 1,
  Color_Blue = 2,
};

inline const char **EnumNamesColor() {
  static const char *names[] = { "Red", "Green", "Blue", nullptr };
  return names;
}

inline const char *EnumNameColor(int e) { return EnumNamesColor()[e]; }
```

枚举和方便的反向查找。

```protobuf
enum {
  Any_NONE = 0,
  Any_Monster = 1,
};

inline const char **EnumNamesAny() {
  static const char *names[] = { "NONE", "Monster", nullptr };
  return names;
}

inline const char *EnumNameAny(int e) { return EnumNamesAny()[e]; }
```

Unit 与枚举有很多共同之处。

```protobuf
struct Vec3;
struct Monster;
```

预先声明所有的数据类型，因为类型之间的循环引用是允许的（但对象之间的循环引用是不允许的）。

```protobuf
FLATBUFFERS_MANUALLY_ALIGNED_STRUCT(4) Vec3 {
 private:
  float x_;
  float y_;
  float z_;

 public:
  Vec3(float x, float y, float z)
    : x_(flatbuffers::EndianScalar(x)), y_(flatbuffers::EndianScalar(y)), z_(flatbuffers::EndianScalar(z)) {}

  float x() const { return flatbuffers::EndianScalar(x_); }
  float y() const { return flatbuffers::EndianScalar(y_); }
  float z() const { return flatbuffers::EndianScalar(z_); }
};
FLATBUFFERS_STRUCT_END(Vec3, 12);
```

这些丑陋的宏做了几件事：它们关闭了编译器通常可能做的任何填充，因为我们手动添加了填充（尽管在这个例子中没有），并且它们强制执行了FlatBuffers选择的对齐。这确保了这个结构的布局在任何编译器和平台上都看起来是一样的。注意这些字段是私有的：这是因为这些字段存储的是小恩典标量，与平台无关（因为这是序列化数据的一部分）。然后EndianScalar进行来回转换，这在目前所有的移动和桌面平台上都是不可行的，而在剩下的几个大endian平台上则是一条机器指令。



## FlexBuffers

FlatBuffers的无模式版本有他们自己的编码，详见这里。

它与上面提到的许多属性相同，即所有的数据都是通过偏移量访问的，所有的标量都是按照自己的大小对齐的，而且所有的数据总是以小恩典格式存储。

一个区别是，FlexBuffers是从前向后构建的，所以子代数据存储在父代数据之前，而数据的根部从最后一个字节开始。

另一个区别是标量数据是以可变的位数（8/16/32/64）存储的。当前的宽度总是由父类决定的，也就是说，如果标量位于一个向量中，向量会一次性决定所有元素的位宽。为一个特定的向量选择最小位宽是编码器自动做的事情，因此用户通常不关心这个问题，尽管意识到这个特性（不要把一个双倍数和一堆字节大小的元素放在同一个向量中）对提高效率是有帮助的。

与FlatBuffers不同的是，只有一种偏移量，它是一个无符号的整数，表示从它的地址（偏移量存储的地方）开始的负方向上的字节数。
