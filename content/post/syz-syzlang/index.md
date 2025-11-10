---
title: syzkaller的syzlang机制探索
description: syzlang
slug: syzlang
date: 2025-09-29 00:00:00+0000
categories:
    - FUZZ
tags:
    - fuzz
    - syzkaller
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# syzkaller syzlang

今天开一个新坑，我们来分析一下syzkaller的syzlang机制，这也是syzkaller设计上非常巧思的一个部分。我们将会看到，一个简单的txt文件，最终是如何被syzkaller构造为一个包含了详尽参数的系统调用并在qemu中执行的。

内容包括：

- syzlang文档以及加载
- 从参数到系统调用
- 从系统调用到程序

## syzlang模板以及加载

### syzlang与类型

syzlang是syzkaller对系统调用的描述性语言，简单来说就是它会告诉你一个系统调用是由哪些参数组成的，会返回什么类型的返回值。syzlang文档集中在`sys/linux`目录下。

syzkaller在`prog/types.go`文件中对`类型`进行了详细的描述。类型包括诸如：

- ConstType、IntType、FlagsType、LenType、ProcType、CsumType，这一部分都是整数类型，但各有其特定的含义。

- PtrType、VmaType，指针类型。

- BufferType，字节缓冲区类型。

- StructType、ArrayType，结构体和数组类型。

- UnionType，联合体类型。

- ResourceType，资源类型，这是我认为十分巧妙的一种类型。资源一方面如同货物一般，它反映出一个系统调用需要哪些资源，能创造哪些资源，另一方面它也是桥梁，它反映了系统调用之间的参数传递与依赖关系。有的系统调用能够创建另一个系统调用所需的资源，这使得不同的系统调用相互关联了起来。syzkaller在创建模糊测试用例时，可以通过资源将系统调用有逻辑地组合起来，形成特殊的语义，而非盲目生成。不过它只能显式反映系统调用之间的依赖组合关系，对于那些没有明面展示在参数关系上的**隐式依赖关系**，它无法识别，这也是目前的学术方向之一。

### 从txt到go

syzlang模板大致长这个样子，可以大致看出各种系统调用的性状：

![img](syzlang.png)

`syz-sysgen`会读取这些文件，然后将其转换成类似AST语法树的结构，随后调用compiler.Compile函数将其转换为syzkaller中定义的[Resource、Syscall、Type](https://github.com/google/syzkaller/blob/master/pkg/compiler/compiler.go#L76)结构，将其序列化并存储到`sys/gen/*.gob.flate`中，后续由fuzzer读取并使用。

除此以外，还会输出以下文件：

`register.go`: 负责在运行时将编译好的 Syzlang 数据文件 (.gob.flate) 注册到 Syzkaller 框架中，使其能够访问。

`defs.h`: C 语言头文件，包含跨架构的宏定义和结构体定义，例如：

```
#define GOOS "linux"

#define SYZ_PAGE_SIZE 4096

struct call_props_t（
```

`syscalls.h`: C 语言头文件，定义一个名为 syscalls 的常量数组，其中包含了每个系统调用的名称、系统调用号（NR）、属性和调用地址。`syz-executor`会读取并执行这个数组中的系统调用。

## 从参数到系统调用

### 参数

参数，即`Arg`结构，这一部分的定义存储在`prog/prog.go`文件中。

```go
type Arg interface {
	Type() Type
	Dir() Dir
	Size() uint64

	validate(ctx *validCtx, dir Dir) error
	serialize(ctx *serializer)
}
```

最基础的`Arg`接口定义中，包含了类型（Type）、方向（Dir，即这个参数是需要写入的还是读出的，抑或是双向的）、大小（Size）。

基础的参数类型一共有6中，分别是：

- ConstArg: 很好理解，它代表一切和数字相关的变量，比如func x(int a)中的a就是一个ConstArg

- PointerArg: 指针类型的变量，它会指向一个地址，这个地址中又会存储一种其他类型的变量。

- DataArg： 一般用作缓冲区，既可以存入也可以写出

- GroupArg：可以理解为结构体或者数组，它一般会是各种类型变量的集合。

- UnionArg：联合体，它也代表各种类型变量的集合，但是只能选取其中之一。

- ResultArg：结果类型的变量，它代表这个变量可以是其他系统调用的返回值。

各个参数类型的具体定义在这里就不详细展开了，大家可以去看源文件，Syzkaller对各种类型的参数的定义很精巧。

每种参数都有对应的`MakeXXArg`方法，用以相关参数类型的构造。例如`MakeConstArg`：

```go
func MakeConstArg(t Type, dir Dir, v uint64) *ConstArg {
	return &ConstArg{ArgCommon: ArgCommon{ref: t.ref(), dir: dir}, Val: v}
}
```

传入参数的类型、方向以及一个uint64类型的整数，我们就可以得到一个ConstArg类型的变量。复杂一点的如`MakeGroupArg`，我们需要传入一个Arg数组，这个数组中包含了结构体每一个成员变量转化成的Arg。

变量是可以多重嵌套的，借用syzlang模板中的例子：

```
bpf$MAP_CREATE(cmd const[BPF_MAP_CREATE], arg ptr[in, bpf_map_create_arg], size len[arg]) fd_bpf_map
```

这是bpf系统调用bpf_map_create的syzlang描述，其中的第二个参数`arg ptr[in, bpf_map_create_arg]`最外层是一个指针类型的变量，随后是第二层：

```
bpf_map_create_arg [
	base		bpf_map_create_arg_base
	bloom_filter	bpf_map_create_arg_bf
] [varlen]
```

这是一个联合体类型的变量，包括base或者bloom_filter类型，我们在构造的时候只能选其中之一，随后是第三层：

```
type bpf_map_create_arg_t[TYPE, KSIZE, VSIZE, MAX, FLAGS, MAP_EXTRA] {
	type			TYPE
	ksize			KSIZE
	vsize			VSIZE
	max			MAX
	flags			FLAGS
	inner			fd_bpf_map[opt]
	node			int32
	map_name		array[const[0, int8], BPF_OBJ_NAME_LEN]
	map_ifindex		ifindex[opt]
	btf_fd			fd_btf[opt]
	btf_key_type_id		btf_opt_type_id
	btf_value_type_id	btf_opt_type_id
	btf_vmlinux_type_id	btf_opt_type_id
	map_extra		MAP_EXTRA
# NEED: value_type_btf_obj_fd should also depend on the map type but AND operators are not yet supported in conditional fields.
	value_type_btf_obj_fd	fd_btf	(if[value[flags] & BPF_F_VTYPE_BTF_OBJ_FD != 0])
	pad1			const[0, int32]	(if[value[flags] & BPF_F_VTYPE_BTF_OBJ_FD == 0])
	map_token_fd		fd_bpf_token	(if[value[flags] & BPF_F_TOKEN_FD != 0])
	pad2			const[0, int32]	(if[value[flags] & BPF_F_TOKEN_FD == 0])
} [packed]

type bpf_map_create_arg_base bpf_map_create_arg_t[flags[bpf_map_type, int32], int32, int32, int32, flags[map_flags, int32], const[0, int64]]
type bpf_map_create_arg_bf bpf_map_create_arg_t[const[BPF_MAP_TYPE_BLOOM_FILTER, int32], int32, int32, int32, flags[map_flags, int32], int64[0:15]]

```

这里的定义十分灵活，第三层的描述是`bpf_map_create_arg_t`，一个GroupArg。但是base和bloom_filter对其中不同参数的类型做了不同的限定。**这使得同一参数应用于不同场景时可以满足正确的约束**。嵌套中仍然可以继续嵌套。

### 系统调用

系统调用，即`Call`：

```go
type Call struct {
	Meta    *Syscall
	Args    []Arg
	Ret     *ResultArg
	Props   CallProps
	Comment string
}
```

它的成员包括系统调用定义（Meta，Meta会记录系统调用相关的信息，如调用名、各种参数类型等）、参数（Args，参数实例数组）、返回值参数、属性（它代表如何控制或修改这次调用）、注释。

## 从系统调用到程序

### 程序

程序就是`Call`的集合：

```go
type Prog struct {
	Target   *Target
	Calls    []*Call
	Comments []string

	// Was deserialized using Unsafe mode, so can do unsafe things.
	isUnsafe bool
}
```

### executor运行

todo

## 实战与结语

这里我将展示一段代码，运用上述逻辑手动生成一个socket系统调用：

```go
func genSocketCall(r *randGen, s *state, domain int, sock_type int, proto int) (call *Call) {
	meta := r.target.SyscallMap["socket"] //获取socket系统调用元数据
	c := MakeCall(meta, nil)
	c.Args, _ = r.generateArgs(s, meta.Args, DirIn)

    // 生成第一个参数
	domainArg := MakeConstArg(meta.Args[0].Type, DirIn, uint64(domain))
	c.Args[0] = domainArg

    // 生成第二个参数
	typeArg := MakeConstArg(meta.Args[1].Type, DirIn, uint64(sock_type))
	c.Args[1] = typeArg

    // 生成第三个参数
	protoArg := MakeConstArg(meta.Args[2].Type, DirIn, uint64(proto))
	c.Args[2] = protoArg

	r.target.assignSizesCall(c)
	return c
}
```

`Syzkaller`作为当前最热门的内核模糊测试工具，具有极强的扩展性，养活了一批学者，还是很有分析的价值的，对syzlang到系统调用转换的分析或许可以帮助我们在以下几个方面做一些有价值的工作：

- 扩充syzlang模板，做开源贡献（SyzDescribe、KernelGpt）

- 生成特定类型的系统调用，集中测试内核的某个部分（BRF）

- 系统调用之间的隐式依赖分析（Moonshine）