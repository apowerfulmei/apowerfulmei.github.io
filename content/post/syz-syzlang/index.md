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
weight: 2       # You can add weight to some posts to override the default sorting (date descending)
---

# syzkaller syzlang

今天开一个新坑，我们来分析一下syzkaller的syzlang机制，这也是syzkaller设计上非常巧思的一个部分。我们将会看到，一个简单的txt文件，最终是如何被syzkaller构造为一个包含了详尽参数的系统调用并在qemu中执行的。

内容包括：

- syzlang文档以及加载
- 从参数到系统调用
- 从系统调用到程序

## syzlang模板

## 从参数到系统调用

### 参数

参数，即`Arg`结构，这一部分的定义存储在`prog/prog.go`文件中。

```
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

```
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

```
type Call struct {
	Meta    *Syscall
	Args    []Arg
	Ret     *ResultArg
	Props   CallProps
	Comment string
}
```

它的成员包括系统调用定义（Meta，Meta会记录系统调用相关的信息，如调用名、各种参数类型等）、参数（Args，参数实例数组）、返回值参数、属性（它代表如何控制或修改这次调用）、注释。