---
title: bpf_link_create Device or resource busy
description: 一次bpf_link_create调试
slug: hello bpf_link_create
date: 2025-11-14 00:00:00+0000
categories:
    - BPF
tags:
    - BPF
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# Error Debugging: Bpf_Link_Create, Device or resource busy

## Debugging Process

Today, I wrote an eBPF program with type `BPF_PROG_TYPE_TRACING` to trace the kernel function `bpf_lsm_file_mprotect` on a virtual machine created by Qemu. 

```
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65528),
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65520),
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65512),
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65504),
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65496),
BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65488),
BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, 0xffffffd0),
BPF_LD_MAP_FD(BPF_REG_1, map_30),
BPF_EMIT_CALL(BPF_FUNC_map_peek_elem),
BPF_LD_IMM64(BPF_REG_0, 0x0),
BPF_EXIT_INSN()
```

I loaded it with `BPF_PROG_LOAD` and tried to use `BPF_LINK_CREATE` to make it attached. But the `link` syscall failed and returned error code 19 which means `Device or resource busy`. That's weird because before that, there was no other program or any thing else being hooked on `bpf_lsm_file_mprotect`. 

However, the program could be linked successfully on the host machine. This gave me some ideas for debugging.

I first tried to figure out who was hooking `bpf_lsm_file_mprotect`. But it seems that there is no command could help me figure out the problem. During this process, I understand the relation between `link` and `ftrace`.

As the program could be linked on the host machine, I thought there must be some difference in kernel config between the host and virtural machine. 

Finally, I found that the kernel config `CONFIG_DYNAMIC_FTRACE` should be enabled. `CONFIG_FUNCTION_TRACER` also needs to be enabled as a dependency.

## Why?

The kernel configuration option **`CONFIG_DYNAMIC_FTRACE`** is **mandatory** for attaching eBPF programs, specifically those using the `fentry` or `fexit` tracing types, to kernel functions.

When enabled, `CONFIG_DYNAMIC_FTRACE` ensures that the compiler inserts a small, **patchable instruction** (initially a `NOP` or "No Operation") at the beginning of every traceable kernel function. This mechanism allows the kernel to **dynamically patch** the function's entry point at runtime.

When an eBPF program attempts to link to a function, the kernel uses this dynamic patching mechanism to **replace the `NOP`** with a jump instruction that redirects control flow to the eBPF helper code. Without this configuration, the necessary **hook points** are absent, leading to errors like **`Device or resource busy`** when attachment is attempted.


## Code

```c
#define _GNU_SOURCE
#include <errno.h>
#include <stdlib.h>
#include <sys/syscall.h>
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <stdio.h>
#include <linux/bpf.h>

#define BPF_STX_MEM(SIZE, DST, SRC, OFF)			\
	((struct bpf_insn) {					\
		.code  = BPF_STX | BPF_SIZE(SIZE) | BPF_MEM,	\
		.dst_reg = DST,					\
		.src_reg = SRC,					\
		.off   = OFF,					\
		.imm   = 0 })

#define BPF_MOV64_REG(DST, SRC)					\
	((struct bpf_insn) {					\
		.code  = BPF_ALU64 | BPF_MOV | BPF_X,		\
		.dst_reg = DST,					\
		.src_reg = SRC,					\
		.off   = 0,					\
		.imm   = 0 })

#define BPF_ALU64_IMM(OP, DST, IMM)				\
	((struct bpf_insn) {					\
		.code  = BPF_ALU64 | BPF_OP(OP) | BPF_K,	\
		.dst_reg = DST,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = IMM })

#define BPF_LD_IMM64_RAW_FULL(DST, SRC, OFF1, OFF2, IMM1, IMM2)	\
	((struct bpf_insn) {					\
		.code  = BPF_LD | BPF_DW | BPF_IMM,		\
		.dst_reg = DST,					\
		.src_reg = SRC,					\
		.off   = OFF1,					\
		.imm   = IMM1 }),				\
	((struct bpf_insn) {					\
		.code  = 0, /* zero is reserved opcode */	\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = OFF2,					\
		.imm   = IMM2 })

/* pseudo BPF_LD_IMM64 insn used to refer to process-local map_fd */

#define BPF_LD_MAP_FD(DST, MAP_FD)				\
	BPF_LD_IMM64_RAW_FULL(DST, BPF_PSEUDO_MAP_FD, 0, 0,	\
			      MAP_FD, 0)

#define BPF_EMIT_CALL(FUNC)					\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_CALL,			\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = ((FUNC) - BPF_FUNC_unspec) })

#define BPF_EXIT_INSN()						\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_EXIT,			\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = 0 })
#define BPF_MOV64_IMM(DST, IMM)					\
	((struct bpf_insn) {					\
		.code  = BPF_ALU64 | BPF_MOV | BPF_K,		\
		.dst_reg = DST,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = IMM })

#define offsetof(type, member)	__builtin_offsetof(type, member)
#define ARRAY_SIZE(x) (sizeof(x) / sizeof(*(x)))


static inline __u64 ptr_to_u64(const void *ptr)
{
	return (__u64) (unsigned long) ptr;
}


int bpf_map_create(uint32_t map_type, uint32_t key_size, uint32_t value_size, unsigned int max_entries, unsigned int flags) {
	union bpf_attr attr = {.map_type = map_type,
		.key_size = key_size,
		.value_size = value_size,
		.max_entries = max_entries,

               };
        if (flags != -1) {
            attr.map_flags = flags;
        } 

	return syscall(SYS_bpf, BPF_MAP_CREATE, &attr, 0x40);
}



int link_create(int prog_fd, int target_fd, uint32_t attach_type, uint32_t flags, uint32_t target_btf_id)
{
        union bpf_attr attr = {
                .link_create.prog_fd = prog_fd,
                .link_create.target_fd = target_fd,
                .link_create.attach_type = attach_type,
                .link_create.flags = 8,
        };

        return syscall(SYS_bpf, BPF_LINK_CREATE, &attr, sizeof(attr.link_create));
}


// loads a prog and returns the FD
int load_prog(struct bpf_insn *instructions, size_t insn_count)
{
        printf("%zu\n", insn_count);
        unsigned char log_buf[1000000] = {};
        memset(log_buf, 0, 1000000);
        union bpf_attr attr ;
        attr.prog_type = BPF_PROG_TYPE_TRACING;
        attr.insns = (uint64_t)instructions;
        attr.insn_cnt = insn_count;
        attr.license = (uint64_t) "GPL";
        attr.log_size = sizeof(log_buf);
        attr.log_buf = (uint64_t)log_buf;
        attr.log_level = 3;
        attr.attach_btf_id = 116106;///*btf_id*/;
        attr.expected_attach_type = BPF_TRACE_FENTRY;
        
        // load the BPF program
        int prog_fd = syscall(SYS_bpf, BPF_PROG_LOAD, &attr, sizeof(attr));
		printf("prog_fd = %d\n", prog_fd);

        if (prog_fd < 0) {
                for (int i = 0; i < sizeof(log_buf) && log_buf[i] != '\0'; i++) {
                        if (log_buf[i] != '\n') {
                                printf("%c",log_buf[i]);
                        } else {
                                printf("\n");
                        }
                }     
                printf("%s\n", strerror(errno));
                // printf("could load program\n");

                return -1;
        }

        return prog_fd;
}

int load_run_bpf_insn()
{

    int map_id=bpf_map_create(BPF_MAP_TYPE_BLOOM_FILTER, 0, 4, 32, -1);

    printf("create map map_30: %d\n", map_id);
    
    
    struct bpf_insn prog[] = {
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65528),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65520),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65512),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65504),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65496),
        BPF_STX_MEM(BPF_DW, BPF_REG_10, BPF_REG_1, 65488),
        BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, 0xffffffd0),
        BPF_LD_MAP_FD(BPF_REG_1, map_id),
        BPF_EMIT_CALL(BPF_FUNC_map_peek_elem),
        BPF_MOV64_IMM(BPF_REG_0, 0x0),
        BPF_EXIT_INSN()

    };
  

        int prog_fd = load_prog(prog, /*prog_len=*/sizeof(prog) / sizeof(prog[0]));
        if ( prog_fd < 0) {
                printf("Could not load program\n");
                return -1;
        }
        int link = link_create(prog_fd, 0, BPF_TRACE_FENTRY, 0, 0);
        int link2 = link_create(prog_fd, 0, BPF_TRACE_FENTRY, 0, 0);
        printf("link created: %d\n", link);
        printf("link2 created: %d\n", link2);
        if(link2<0){
                perror("link2 create failed");
                int err=errno;
                printf("error code: %d\n", err);
        }
}


int main(int argc, char **argv)
{
	load_run_bpf_insn();
}


```

## Referrence

https://www.kernelconfig.io/index.html