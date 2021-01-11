---
title: "在Go Module 下使用 gogoprotobuf 的引用问题"
date: "2021-01-07"
toc: "true"
---

protobuf 大家应该不会陌生，虽然当前官方已经发布了 protobuf proto3版本，增加了许多功能性的特性，但在序列化的效率上似乎仍不如 gogoprotobuf，gogoprotobuf 仍占有着很多的市场，今天基于 gogoprotobuf 探讨几个在 Go Module 项目下的 proto 间引用的相关问题。

在使用 Go Module 时，考虑以下目录结构，假设 Go Module 初始化为 prototest

```
prototest
├── go.mod
├── go.sum
├── main.go
├── protos
│   ├── aproto
│   │   ├── m
│   │   │   └── m.proto
│   │   ├── n
│   │   │   └── n.proto
│   │   ├── a.proto
│   │   └── b.proto
│   └── bproto
│       └── p.proto
```

### 同包引用

如果在同一个包下包含两个 .proto 文件并存在引用，如 a.proto import b.proto，那么在 a.proto 中需要写入 import 语句，protobuf 会根据 .proto 文件中的 import 路径来生成 .pb.go 文件中的 import 路径，而在实际的项目中，a.pb.go 和 b.pb.go 实际上是同一个包的文件，同一包内的文件是不需要使用 import 来引用的，如果写入全路径的 import 会编译不通过：“无法引用自身”。

这就产生了矛盾:  .proto 文件需要写 import，生成后根据该 import 在 .pb.go 中写入 import， 而 .pb.go 又不应该写 import

Gogoproto 有两个大版本 1.2.x 和 1.3.x

- 在 1.3.x 版本的插件里，这个问题好像没有什么解决方案，只能将存在引用的 .proto 文件拆分为不同的包，包与包之间可以进行引用

- 在 1.2.x 版本的插件里，有一种方法可以在生成 .pb.go 文件时不写入 import，在 .proto 文件中 import 时只使用文件名如 import "b.proto"，只要保证在生成时 `--proto_path` 配置正确能够找到 b.proto， 那么在生成后的 .pb.go 文件中便认为没有包引入，因此就没有 import 写入。这种方法在 1.3.x 版本使用时会在 .pb.go 中写入 `import "."`，依旧无法通过编译，这一点反倒是兼容性上的退步（可能有更好的解决方案我还没发掘到，如有了解欢迎讨论）

### 跨包引用

讨论完同包引用的问题，再说一下跨包引用，假设现在 bproto 中的 p.proto 需要引用 aproto 中的 a.proto

前面提到了包与包之间是可以引用的，但这里还存在几个需要注意的问题：

- 第一个是我们之前提到的，要保证生成后的 .pb.go 文件适配整个项目的包路径，保证编译通过。那么在生成的 p.pb.go 中，应存在 `import aproto1 "prototest/protos/aproto"`，而 p.proto 中就应写为`import "prototest/protos/aproto/b.proto"`，这样一来想要找到被 import 的文件的话，就要求实际的路径与 Go Module 包的路径完全一致，才能正确配置 `--proto_path` 以找到引用。通常情况下 prototest 就是项目的根目录了，这意味着我们可能需要使用到涉及项目外部的路径。

- 第二个是 go protobuf 生成对于 import 的识别是完全基于 .proto 文件中的 import 路径的。在当前 p.proto 引用 a.proto 的前提下，我们还需要考虑另一个点，a.proto 还引用了 b.proto。proto 的生成不像编译，import 的查找阶段还是比较简单原始，如果  `--proto_path` 的配置找不到 b.proto ，那么 import 就会失败，因此如果有多级引用，就必须将所有引用的路径全部配置到  `--proto_path` 中。

- 第三个是额外的一种情况，如果是上一个问题中提到的情况，存在同包引用，那么 a.proto 中引用 b.proto 写入的是`import "b.proto"`，而如果此时 p.proto 也同时需要引用 b.proto，即 p.proto 同时引用了 a.proto 和 b.proto，那么 b.proto 会存在两个 import 路径，一个是在 a.proto 中的`import "b.proto"`，另一个是在 p.proto 中的`import "prototest/protos/aproto/b.proto"` 。protobuf 是无法识别这是同一个文件的，当你在 `--proto_path` 中配置好引用能同时找到两个路径时，它会告诉你出现了重复的定义。因此你必须保证所有文件的每个引用写入的引用路径是相同的，这样才能被识别为同一个文件。

### 如何解决？

因为历史原因我们的项目出现了上述的所有问题：同包引用、跨包引用、同包 + 跨包引用

先说同包引用的问题：其实最好的解决方案是将同包引用全部拆分为跨包引用，即将 a.proto、b.proto 全部改为 m.proto、n.proto 的形式，这样一来全部的引用路径都是完整的包路径，也不会出现同一个文件被识别为两个文件造成重复定义的问题。如果文件太多不方便拆分，那么就只能采用使用低版本然后不写路径同包引用的方法。这种情况下还要处理重复识别的情况，我们的做法是在生成 bproto 包时，将 aproto 包中的文件复制并修改其引用路径为完整包路径，单独用于给 bproto 提供引用使用，从而保证生成时引用的路径唯一。由于在生成 bproto 的文件时并不会去链接 aproto 的 .pb.go 文件，因此不需要担心 aproto 在修改为完整包路径后出现的引用自身错误情况。

再说一下生成后 .pb.go 文件的包路径适配问题：刚才也提到了我们会将 aproto 中的文件做复制来作为引用的文件，实际上为了保证 import 路径与配置的 `--proto_path` 查找引用路径一致，我们会生成一个临时目录来单独存放复制来的引用文件，而该目录的全路径便是包的全路径。这一方式的好处是有时某些项目的完整包路径与项目目录实际路径并不一样，而创建临时目录便可以符合引入路径的需求。

### 总结

总的来说，使用 Go Module 时，protobuf 在引用问题上还是很令人头疼的，但好在通过处理还是能有效的解决问题。因此在最初构建项目架构时一定要做好充足的考虑，也希望在后面 protobuf 能更好的处理引用问题。



