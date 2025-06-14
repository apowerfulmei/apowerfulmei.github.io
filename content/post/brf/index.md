---
title: BRF复现
description: BRF复现
slug: hello brf
date: 2025-06-14 00:00:00+0000
categories:
    - Fuzz
tags:
    - Fuzz
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# BRF

## 简介

[BRF: Fuzzing the eBPF Runtime](https://dl.acm.org/doi/abs/10.1145/3643778) 是一个针对ebpf runtime组件的模糊测试工具。它旨在解决syzkaller对ebpf模糊测试低效的问题，从程序语义、程序依赖、程序执行三个要点出发，生成了有高verifier通过率、以及丰富语义（即包含了一些列helper与map操作）的C程序，并与syzkaller结合，对ebpf进行模糊测试，大大提升了代码覆盖率，并发现了6个新的ebpf漏洞。

仓库地址：https://github.com/trusslab/brf

## 复现

### 基础复现流程

BRF仓库的README文件提供了详细的复现流程，可以直接按照这个流程走下来。不过在这个过程中，我仍然遇到了一些难以解决的问题。

### BRF编译问题

在make BRF源代码时，会产生报错

## 微调流程


## 微调后的大模型使用


## 参考