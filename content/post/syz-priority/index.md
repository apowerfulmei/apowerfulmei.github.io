---
title: "Syzkaller 的 Priority 机制：从 Signal 到变异的四层优先级体系"
description: "深入分析 syzkaller 内核模糊测试框架中的四层优先级体系——Signal Priority、Corpus 程序优先级、ChoiceTable 系统调用优先级矩阵、参数变异优先级，以及它们如何协同构成正反馈闭环"
slug: syz-priority
date: 2026-04-03 00:00:00+0000
categories:
    - Fuzz
tags:
    - fuzz
    - syzkaller
    - kernel
weight: 2
---

# Syzkaller 的 Priority 机制：从 Signal 到变异的四层优先级体系

文章使用**WorkBuddy**撰写。

## 引言：为什么 syzkaller 需要 priority？

在上一篇[队列机制](/syz-queue/)的分析中，我们看到 syzkaller 通过五级优先级队列解决了"任务调度顺序"的问题——triage 请求优先于普通 fuzz，新候选程序优先于 corpus 程序。

然而，队列机制只回答了"先执行哪个任务"，却没有回答另一组同样关键的问题：

- **一个 syscall 执行成功返回了新的覆盖信息，这条信息有多重要？**
- **corpus 里有上千个程序，变异时该选哪个？**
- **生成新程序时，下一个 syscall 该选哪个？**
- **选定一个 call 后，该变异它的哪个参数？**

这些问题构成了 syzkaller 的 **priority 体系**——一套贯穿 signal → corpus → syscall 选择 → 参数变异的四层优先级机制。与队列机制的"宏观调度"不同，priority 体系解决的是"微观资源分配"：在有限的执行预算内，将 fuzzing 的火力集中在最可能发现新覆盖的方向上。

本文将从源码出发，自底向上逐层拆解这四层优先级。

```
┌─────────────────────────────────────────────────────────────┐
│  第四层：参数变异优先级（getMutationPrio）                    │
│  选定 call 后，按参数类型分配 0~10 的变异权重                │
├─────────────────────────────────────────────────────────────┤
│  第三层：ChoiceTable（prio.go）                              │
│  syscall 间优先级矩阵，指导生成/变异时选哪个 syscall          │
├─────────────────────────────────────────────────────────────┤
│  第二层：Corpus 程序优先级（saveProgram / chooseProgram）     │
│  signal 数量 = 被选中变异的概率，形成正反馈闭环              │
├─────────────────────────────────────────────────────────────┤
│  第一层：Signal Priority（signalPrio）                      │
│  每条覆盖信号带 0~3 的权重，决定哪些 signal "算新的"         │
└─────────────────────────────────────────────────────────────┘
```

---

## 第一层：Signal Priority——覆盖信号的权重标记

### 核心数据结构

在 syzkaller 中，"signal"（信号）代表程序执行产生的覆盖信息。它的底层实现是一个 map，每个元素关联一个 `prioType` 优先级：

```go
// pkg/signal/signal.go

type (
    elemType uint64
    prioType int8
)

// Signal: 覆盖元素到优先级的映射
type Signal map[elemType]prioType
```

这不是一个简单的"有/无"集合，而是每个覆盖元素都携带了**优先级标记**。这个优先级直接影响哪些 signal 会被认定为"新的"。

### signalPrio：四种优先级等级

`signalPrio()` 函数为每次 syscall 执行结果计算一个 0~3 的优先级值：

```go
// pkg/fuzzer/fuzzer.go

func signalPrio(p *prog.Prog, info *flatrpc.CallInfo, call int) (prio uint8) {
    if call == -1 {
        return 0                               // Extra call，最低优先级
    }
    if info.Error == 0 {
        prio |= 1 << 1                         // 调用成功 → +2
    }
    if !p.Target.CallContainsAny(p.Calls[call]) {
        prio |= 1 << 0                         // 不含 any 参数 → +1
    }
    return
}
```

两个独立的维度组合产生四个等级：

| 条件 | prio 值 | 含义 |
|------|---------|------|
| call == -1 | 0 | Extra call，最低价值 |
| 调用失败（errno != 0）或含 any | 0 | 调用失败，且参数含低信息量数据 |
| 调用成功 + 含 any | 2 | 调用成功，但参数含 any |
| 调用失败 + 不含 any | 1 | 调用失败，但参数有具体语义 |
| 调用成功 + 不含 any | **3** | 调用成功 + 参数语义完整，**最高价值** |

设计意图值得细说：

1. **"成功调用比失败调用更有价值"**：`errno == 0` 意味着 syscall 顺利走完了内核路径。失败调用往往在入口处就返回了，覆盖的内核路径较浅。因此成功的调用被赋予了更高的 signal 优先级。

2. **"不含 any 的调用更有价值"**：syzkaller 中的 `any` 类型是一种"万能占位符"，在不知道参数具体含义时用作默认值。含 any 的参数缺少语义信息——我们不知道这份数据"意味着什么"，它覆盖的代码路径可能只是碰巧经过。不含 any 意味着参数有具体的类型和语义（如 `fd[sock]`、`flags[O_RDONLY]`），这样的覆盖更具结构性。

### DiffRaw：只有"更好的" signal 才算新 signal

当一次执行产生覆盖信息后，需要判断这些覆盖中哪些是"新的"。`DiffRaw` 方法实现了这一判断：

```go
// pkg/signal/signal.go

func (s Signal) DiffRaw(raw []uint64, prio uint8) Signal {
    var res Signal
    for _, e := range raw {
        // 关键条件：只有当新 signal 的 prio > 已有 signal 的 prio 时，才算"新的"
        if p, ok := s[elemType(e)]; ok && p >= prioType(prio) {
            continue                              // 已有同等或更高优先级的覆盖，跳过
        }
        if res == nil {
            res = make(Signal)
        }
        res[elemType(e)] = prioType(prio)
    }
    return res
}
```

这意味着：如果某个覆盖元素已经被一个成功的、不含 any 的调用（prio=3）覆盖过，那么同一个元素被一个失败的调用（prio=0）再次产生时，**不会**被认定为新 signal。只有当新的覆盖带有**更高优先级**时，才会更新已有的记录。

### addRawMaxSignal：层层递进的信号收集

在 fuzzer 主循环中，每次执行完成后调用 `addRawMaxSignal` 收集新 signal：

```go
// pkg/fuzzer/cover.go

func (cover *Cover) addRawMaxSignal(signal []uint64, prio uint8) signal.Signal {
    cover.mu.Lock()
    defer cover.mu.Unlock()
    diff := cover.maxSignal.DiffRaw(signal, prio)  // 与历史最大 signal 比较
    if diff.Empty() {
        return diff                                  // 没有新 signal
    }
    cover.maxSignal.Merge(diff)                     // 更新历史最大 signal
    cover.newSignal.Merge(diff)                     // 记录本轮新增 signal
    return diff
}
```

这里维护了两份 signal：`maxSignal`（历史峰值，包括 flaky 的覆盖）和 `newSignal`（本轮新增）。前者用于"去重判断"，后者用于触发 triage 等后续流程。

---

## 第二层：Corpus 程序优先级——signal 数量决定被选中的概率

### 权重计算：signal 越多越容易被选中

当一个程序被发现携带新 signal 并进入 corpus 时，它的"被选中概率"由它携带的 signal 数量决定：

```go
// pkg/corpus/prio.go

func (pl *ProgramsList) saveProgram(p *prog.Prog, signal signal.Signal) {
    prio := int64(len(signal))          // signal 数量即为优先级
    if prio == 0 {
        prio = 1                        // 至少为 1，确保不会被完全忽略
    }
    pl.sumPrios += prio
    pl.accPrios = append(pl.accPrios, pl.sumPrios)  // 累积前缀和
    pl.progs = append(pl.progs, p)
}
```

这是一个直觉上非常合理的设计：一个程序覆盖的 signal 越多，它在 corpus 中的"代表性"越强，越值得被选中进行变异。

### 加权随机选择：累积前缀和 + 二分查找

`chooseProgram` 使用经典的**前缀和 + 二分查找**算法实现加权随机选择：

```go
// pkg/corpus/prio.go

func (pl *ProgramsList) chooseProgram(r *rand.Rand) *prog.Prog {
    if len(pl.progs) == 0 {
        return nil
    }
    randVal := r.Int63n(pl.sumPrios + 1)           // 生成 [0, sumPrios] 的随机值
    idx := sort.Search(len(pl.accPrios), func(i int) bool {
        return pl.accPrios[i] >= randVal            // 二分查找第一个 >= randVal 的位置
    })
    return pl.progs[idx]
}
```

举例说明：假设 corpus 中有三个程序，分别携带 2、5、3 个 signal。累积前缀和为 `[2, 7, 10]`。生成的随机值落在 `[0,2]` 时选中第一个程序，落在 `[3,7]` 时选中第二个，落在 `[8,10]` 时选中第三个——选择概率严格正比于 signal 数量（20%、50%、30%）。

测试用例 `TestChooseProgram` 验证了这一性质：

```go
// pkg/corpus/prio_test.go
prob := float64(prio) / float64(corpus.sumPrios)
diff := math.Abs(prob*maxIters - float64(counters[p]))
if diff > eps*maxIters {
    t.Fatalf("the difference (%f) is higher than %f%%", diff, eps*100)
}
```

### 正反馈闭环

这层优先级构成了一条**正反馈闭环**：

```
程序 A 覆盖更多 signal
       ↓
saveProgram(A, signal) → A 在 corpus 中获得更高权重
       ↓
chooseProgram() 更频繁地选中 A 进行变异
       ↓
A 的变异后代更可能发现新 signal
       ↓
新 signal 再次进入 corpus → 闭环
```

这条闭环确保了 fuzzer 的火力集中在"高产"程序上，而非均匀分配。但也引入了风险——如果某个程序覆盖的都是"浅层" signal，反复变异它可能导致探索停滞。syzkaller 通过 ChoiceTable 的 5% 随机逃逸机制（后文详述）来缓解这一问题。

### Focus Area：引导 fuzzer 探索特定代码区域

除了全局加权选择外，syzkaller 还支持 **Focus Area** 机制——允许用户指定感兴趣的区域，引导 fuzzer 优先探索这些区域：

```go
// pkg/corpus/corpus.go

type FocusArea struct {
    Name     string              // 区域名称（可空）
    CoverPCs map[uint64]struct{} // 该区域关联的覆盖 PC
    Weight   float64             // 权重
}
```

当设置了 Focus Area 时，`ChooseProgram` 先按权重选择区域，再在区域内按 signal 加权选择程序：

```go
// pkg/corpus/prio.go

func (corpus *Corpus) ChooseProgram(r *rand.Rand) *prog.Prog {
    // ...
    var randArea *focusAreaState
    if len(corpus.focusAreas) > 0 {
        sum := 0.0
        for _, area := range corpus.focusAreas {
            sum += area.Weight
        }
        val := r.Float64() * sum                // 按权重随机选区域
        // ... 遍历找到对应的区域
    }
    if randArea != nil {
        return randArea.chooseProgram(r)        // 在区域内按 signal 加权选
    }
    return corpus.chooseProgram(r)              // 无 Focus Area 时全局选
}
```

测试 `TestFocusAreas` 验证了三个权重为 10:30:60 的 Focus Area，选择比例严格满足 10%:30%:60%。

### Corpus.Save 的去重与合并

当一个程序已存在于 corpus 中（相同的序列化签名），新的 signal 会被 **合并**到已有 Item，而非创建新条目：

```go
// pkg/corpus/corpus.go (简化)

func (corpus *Corpus) Save(inp *prog.Input) {
    sig := string(inp.Prog.Serialize())
    if old, exists := corpus.progsMap[sig]; exists {
        // 合并新 signal 到已有 Item
        newSignal := old.Signal
        newSignal.Merge(inp.Signal)
        // ... 更新 Item
    } else {
        // 新程序，创建新 Item 并 saveProgram
        corpus.saveProgram(inp.Prog, inp.Signal)
    }
}
```

---

## 第三层：ChoiceTable——系统调用间优先级矩阵

如果说前两层决定了"选哪个程序"，那么 ChoiceTable 决定的是"在已有一个 syscall 的上下文中，下一个该选什么 syscall"。这是一张 **N×N 的优先级矩阵**，其中 N 是可用 syscall 的数量。

### 设计动机

考虑这样一个场景：程序中已经有了一个 `open("/tmp/file")` 调用。下一个 syscall 选什么更有可能触发新覆盖？

直觉上，选 `read(fd)` 比选 `getpid()` 更有价值——因为 `open` 产生了一个文件描述符资源，而 `read` 消费这个资源，两个 syscall 的组合更有可能探索到深层内核路径。

syzkaller 将这种"谁跟谁配对更好"的直觉量化为一张优先级矩阵。矩阵的每个元素 `prio[i][j]` 表示：在已知有 syscall i 的上下文中，选择 syscall j 的优先级有多高。

### ChoiceTable 结构体

```go
// prog/prio.go

type ChoiceTable struct {
    target *Target
    runs   [][]int32        // 累积前缀和数组（用于快速二分查找）
    calls  []*Syscall       // 可生成的 syscall 列表
}
```

`runs` 是核心数据结构。对于每一行 `runs[i]`，它存储了 `prio[i][0..N-1]` 的累积前缀和。查找时使用二分搜索，时间复杂度 O(log N)。

### 静态优先级：基于参数类型的资源分析

静态优先级是在编译期（更准确地说，是在初始化时）通过分析所有 syscall 的参数类型来计算的。

#### 第一步：calcResourceUsage——提取资源使用关系

`calcResourceUsage` 遍历所有 syscall 的所有参数类型，记录每个 syscall 对各类资源的使用情况：

```go
// prog/prio.go

func (target *Target) calcResourceUsage(enabled map[*Syscall]bool) map[string]map[int]weights {
    uses := make(map[string]map[int]weights)
    ForeachType(target.Syscalls, func(t Type, ctx *TypeCtx) {
        switch a := t.(type) {
        case *ResourceType:
            // 记录对 fd[sock] 等资源类型的使用
            str := "res"
            for i, k := range a.Desc.Kind {
                str += "-" + k
                w := int32(10)
                if i < len(a.Desc.Kind)-1 {
                    w = 2                              // 非最具体的层级权重低
                }
                noteUsage(uses, c, w, ctx.Dir, str)
            }
        case *PtrType:
            if _, ok := a.Elem.(*StructType); ok {
                noteUsagef(uses, c, 10, ctx.Dir, "ptrto-%v", a.Elem.Name())
            }
            // ... UnionType, ArrayType 类似处理
        case *BufferType:
            if a.Kind == BufferFilename {
                noteUsage(uses, c, 10, DirIn, "filename")  // 文件名是强关联
            }
        case *VmaType:
            noteUsage(uses, c, 5, ctx.Dir, "vma")          // 内存地址权重中等
        }
    })
    return uses
}
```

每个记录包含 `in`（只读使用）和 `inout`（读写使用）两个权重。例如，`open` 的第一个参数是 `filename`（buffer），`dir=DirIn`；它的返回值是 `fd`（resource），`dir=DirOut`。

#### 第二步：基于使用关系计算优先级

```go
// prog/prio.go

for _, w0 := range weights {
    for _, w1 := range weights {
        if w0.call == w1.call {
            continue
        }
        // 核心公式：生产者→消费者方向优先级更高
        prios[w0.call][w1.call] += w0.inout*w1.in*3/2 + w0.inout*w1.inout
    }
}
```

这个公式的设计非常精巧：

- `w0.inout * w1.in * 3/2`：c0 产生（inout）某个资源，c1 消费（in）该资源 → **高优先级**。额外乘以 3/2 偏向"生产→消费"方向。
- `w0.inout * w1.inout`：两者都读写该资源 → 中等优先级（互相影响）。
- `w0.in * w1.in`：两者只读 → **不计算**（没有资源流动）。

以 `open` 和 `read` 为例：
- `open` 的输出包含 `fd`（resource, `inout` 权重 10）
- `read` 的输入包含 `fd`（resource, `in` 权重 10）
- `prio[open][read] += 10 * 10 * 3/2 + 10 * 10 = 150 + 100 = 250`

反过来，`read` 的输出不含资源，因此 `prio[read][open]` 会低得多。

#### 第三步：自优先级 = 最大值 × 3/4

```go
// prog/prio.go

for c := range enabled {
    max := slices.Max(prios[c.ID])
    if max == 0 {
        prios[c.ID][c.ID] = 1
    } else {
        prios[c.ID][c.ID] = max * 3 / 4
    }
}
```

自优先级（即连续两次调用同一个 syscall）设为该行最大值的 3/4。既不太高（避免无限重复同一调用），也不太低（连续调用同一 syscall 有时是合理的，如多次 `write`）。

测试 `TestStaticPriorities` 验证了这一点：对 `open → {read, write, mmap}` 组合，`open` 被选为后继的概率显著高于反向。

### 动态优先级：基于 corpus 共现频率

除了静态分析，syzkaller 还统计 corpus 中每个 syscall 对的共现频率：

```go
// prog/prio.go

func (target *Target) calcDynamicPrio(corpus []*Prog, enabled map[*Syscall]bool) [][]int32 {
    prios := make([][]int32, len(target.Syscalls))
    // ... 初始化
    for _, p := range corpus {
        for idx0, c0 := range p.Calls {
            for _, c1 := range p.Calls[idx0+1:] {
                prios[c0.Meta.ID][c1.Meta.ID]++     // 统计共现次数
            }
        }
    }
    for i := range prios {
        for j, val := range prios[i] {
            // 使用 sqrt() 压缩高计数的影响
            // "出现 50 次和 100 次没区别"
            prios[i][j] = int32(2.0 * math.Sqrt(float64(val)))
        }
    }
    normalizePrios(prios, len(enabled))
    return prios
}
```

动态优先级的核心洞察是：**"某些 syscall 经常一起出现，比它们出现了多少次更重要"**。因此使用 `sqrt()` 压缩计数——共现 50 次和 100 次的差异被大幅缩小，避免高频出现的 syscall 对完全主导选择。

### 归一化：|N| × 10 的预算分配

```go
// prog/prio.go

func normalizePrios(prios [][]int32, n int) {
    total := 10 * int32(n)               // 每行总预算 = |N| × 10
    for _, prio := range prios {
        sum := int32(0)
        for _, p := range prio {
            sum += p
        }
        if sum == 0 {
            continue
        }
        for i, p := range prio {
            prio[i] = p * total / sum     // 按比例缩放
        }
    }
}
```

归一化确保每一行的优先级总和为 `|N| × 10`（N 为可用 syscall 数量）。这意味着：即使有 400 个 syscall，每个 syscall 平均仍有 10 个"优先级点数"，保证了选择的基本均匀性。

### BuildChoiceTable：合并静态与动态

```go
// prog/prio.go

func (target *Target) BuildChoiceTable(corpus []*Prog, enabled map[*Syscall]bool) *ChoiceTable {
    prios, enabledCalls := target.CalculatePriorities(corpus, enabled)
    // ... 构建累积前缀和数组
    run := make([][]int32, len(target.Syscalls))
    for i := range run {
        run[i] = make([]int32, len(target.Syscalls))
        var sum int32
        for j := range run[i] {
            if enabledSlice[j] {
                sum += prios[i][j]
            }
            run[i][j] = sum               // 累积前缀和
        }
    }
    return &ChoiceTable{target, run, generatableCalls}
}
```

静态优先级和动态优先级简单相加后归一化，构成最终矩阵。`CalculatePriorities` 中的合并逻辑：

```go
// prog/prio.go

static := target.calcStaticPriorities(enabled)
if len(corpus) != 0 {
    dynamic := target.calcDynamicPrio(corpus, enabled)
    for i, prios := range dynamic {
        for j, p := range prios {
            static[i][j] += p               // 简单相加
        }
    }
}
```

### choose：5% 随机逃逸

`choose` 方法实现了最终的 syscall 选择：

```go
// prog/prio.go

func (ct *ChoiceTable) choose(r *rand.Rand, bias int) int {
    if r.Intn(100) < 5 {
        // 5% 概率完全随机选择，保证探索性
        return ct.calls[r.Intn(len(ct.calls))].ID
    }
    if bias < 0 {
        bias = ct.calls[r.Intn(len(ct.calls))].ID  // 无偏时随机选一个起点
    }
    run := ct.runs[bias]
    runSum := int(run[len(run)-1])
    x := int32(r.Intn(runSum) + 1)
    res := sort.Search(len(run), func(i int) bool {
        return run[i] >= x                           // 二分查找
    })
    return res
}
```

5% 的完全随机选择是一个关键的**逃逸机制**：无论优先级矩阵如何分布，始终有 5% 的概率跳出当前偏好。这防止了 fuzzer 在某些"看似高产实则已经饱和"的 syscall 对上浪费资源。

### ChoiceTable 的动态更新

ChoiceTable 不是一成不变的，它会随着 corpus 的增长而更新：

```go
// pkg/fuzzer/fuzzer.go

func (fuzzer *Fuzzer) ChoiceTable() *prog.ChoiceTable {
    progs := fuzzer.Config.Corpus.Programs()
    // 小 corpus 每 33 个程序更新一次，大 corpus 每 333 个
    regenerateEveryProgs := 333
    if len(progs) < 100 {
        regenerateEveryProgs = 33
    }
    if fuzzer.ctProgs+regenerateEveryProgs < len(progs) {
        select {
        case fuzzer.ctRegenerate <- struct{}{}:     // 异步触发重建
        default:
            // 已有重建在进行，跳过
        }
    }
    return fuzzer.ct
}
```

更新遵循**只增不减原则**：

```go
// pkg/fuzzer/fuzzer.go

func (fuzzer *Fuzzer) updateChoiceTable(programs []*prog.Prog) {
    newCt := fuzzer.target.BuildChoiceTable(programs, fuzzer.Config.EnabledCalls)
    fuzzer.ctMu.Lock()
    defer fuzzer.ctMu.Unlock()
    if len(programs) >= fuzzer.ctProgs {           // 只有更大才替换
        fuzzer.ctProgs = len(programs)
        fuzzer.ct = newCt
    }
}
```

重建通过独立的 goroutine 异步执行（`choiceTableUpdater`），不阻塞 fuzzing 主循环。

---

## 第四层：参数变异优先级——类型驱动变异资源分配

前三层回答了"选哪个程序"和"选哪个 syscall"，当变异操作进行时，还需要回答：**选哪个参数来变异？**

### 变异操作权重

syzkaller 定义了五种变异操作，每种有固定的权重：

```go
// prog/mutation.go

var DefaultMutateOpts = MutateOpts{
    SquashWeight:     50,
    SpliceWeight:     200,       // 最高权重
    InsertWeight:     100,
    MutateArgWeight:  100,
    RemoveCallWeight: 10,        // 最低权重
}
```

选择时按总权重 460 随机抽样：Splice（拼接两个程序）占比 43.5%，Insert（插入新调用）和 MutateArg（变异参数）各占 21.7%，Squash（压缩）占 10.9%，Remove（删除调用）仅占 2.2%。

Splice 占比最大的原因是：将两个已知有价值的程序拼接在一起，是最有可能触发新覆盖的变异策略。

### chooseCall：按参数复杂度选择要变异的调用

当操作类型为 `MutateArg` 时，首先需要选择程序中的哪个 call 来变异：

```go
// prog/mutation.go

func chooseCall(p *Prog, r *randGen) int {
    var prioSum float64
    var callPriorities []float64
    for _, c := range p.Calls {
        var totalPrio float64
        ForeachArg(c, func(arg Arg, ctx *ArgCtx) {
            prio, stopRecursion := arg.Type().getMutationPrio(
                p.Target, arg, false, c.Meta.Attrs.KFuzzTest)
            totalPrio += prio
            ctx.Stop = stopRecursion
        })
        prioSum += totalPrio
        callPriorities = append(callPriorities, prioSum)
    }
    if prioSum == 0 {
        return -1
    }
    return sort.SearchFloat64s(callPriorities, prioSum*r.Float64())
}
```

每个 call 的"变异价值"等于其所有参数的 `getMutationPrio` 之和。参数类型越复杂、变异空间越大，该 call 被选中变异的概率越高。

### getMutationPrio：各类型参数的变异优先级

`getMutationPrio` 为每种参数类型返回一个 `float64` 优先级，定义了三个关键常量：

```go
const (
    maxPriority = float64(10)   // 最高优先级
    minPriority = float64(1)    // 最低（非零）优先级
    dontMutate  = float64(0)    // 不变异
)
```

各类型的优先级如下：

```go
// prog/mutation.go

// IntType: 整数类型
func (t *IntType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    plainPrio := math.Log2(float64(t.TypeBitSize())) + 0.1*maxPriority  // ~4.3 + 1.0 = 5.3
    if t.Kind != IntRange {
        return plainPrio, false
    }
    size := t.RangeEnd - t.RangeBegin + 1
    switch {
    case size <= 15:
        prio = rangeSizePrio(size)              // 小范围，值得穷举
    case size <= 256:
        prio = maxPriority                       // ≤256 个值，最高优先级
    default:
        prio = plainPrio                         // 大范围等同于普通整数
    }
    return prio, false
}

// StructType: 结构体（有 generator 时）
func (t *StructType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    if target.SpecialTypes[t.Name()] == nil || ignoreSpecial {
        return dontMutate, false                // 普通 struct 不变异
    }
    return maxPriority, true                    // 有 generator 的 struct 整体替换，优先级最高
}

// UnionType: 联合体
func (t *UnionType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    // 单选项 union 或非特殊类型 → 不变异
    // 多选项特殊 union → maxPriority（切换分支可触发不同内核路径）
    // 多选项普通 union → maxPriority（继续递归变异子选项）
}

// FlagsType: 标志位
func (t *FlagsType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    prio = rangeSizePrio(uint64(len(t.Vals)))   // 按取值数量线性增长
    if t.BitMask {
        prio += 0.1 * maxPriority               // BitMask 额外加 10% bonus
    }
    return prio, false
}

// BufferType: 缓冲区
func (t *BufferType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    if t.Kind == BufferCompressed {
        return maxPriority, false                // 磁盘镜像等复杂数据 → 最高优先级
    }
    return 0.8 * maxPriority, false             // 普通 buffer → 8.0
}

// PtrType: 指针
func (t *PtrType) getMutationPrio(...) (prio float64, stopRecursion bool) {
    return 0.3 * maxPriority, false             // 指针本身 → 3.0
}

// ConstType, CsumType: 常量和校验和
// → dontMutate (0)
```

### rangeSizePrio：小范围值的变异优先级

```go
// prog/mutation.go

func rangeSizePrio(size uint64) (prio float64) {
    switch size {
    case 0:
        prio = dontMutate
    case 1:
        prio = minPriority                      // 只有一个值 → 无需变异
    default:
        // 线性增长 + 上限封顶
        prio = math.Min(float64(size)/3+0.4*maxPriority, 0.9*maxPriority)
    }
    return prio
}
```

这个函数的设计体现了一个实用主义原则：`size/3 + 0.4 × 10`，即基础分 4.0 加上按值数量线性增长的部分。当 `size=15` 时，优先级为 `5 + 4 = 9`，接近上限 `0.9 × 10 = 9`。超过 15 以后优先级封顶——因为 `FlagsType` 的注释说"大多数 syscall 的 flag 取值不超过 15 个"，再高也没必要增加优先级了。

### 完整优先级汇总表

| 参数类型 | 优先级 | 设计理由 |
|----------|--------|----------|
| StructType（有 generator） | **10** | 整体替换效果最好，`stopRecursion=true` |
| UnionType（多选项） | **10** | 切换分支可触发完全不同的内核路径 |
| ArrayType（变长） | **10** | 变异数组长度和元素 |
| BufferCompressed | **10** | 磁盘镜像等复杂数据，变异价值最高 |
| IntType（Range ≤ 256） | **10** | 有限范围值得尝试所有值 |
| BufferType（普通） | **8.0** | blob 数据变异 |
| IntType（普通） | ~5.3 | `log₂(bitsize) + 1.0`，64 位整数约 7.0 |
| ResourceType / VmaType / ProcType | **5.0** | 替换资源句柄、内存地址 |
| PtrType | **3.0** | 指针变异中等价值 |
| LenType | **1.0** | 长度修改通常产生"不正确"的结果 |
| ConstType / CsumType | **0** | 不可变 |
| SpecialPointer | **0** | 暂不支持变异 |

### chooseArg：参数级加权选择

选定 call 后，还需要在 call 的所有参数中选一个来变异。`chooseArg` 使用与前述完全相同的累积和 + 二分查找模式：

```go
// prog/mutation.go

func (ma *mutationArgs) chooseArg(r *rand.Rand) (Arg, ArgCtx) {
    goal := ma.prioSum * r.Float64()
    chosenIdx := sort.Search(len(ma.args), func(i int) bool {
        return ma.args[i].priority >= goal
    })
    return ma.args[chosenIdx].arg, ma.args[chosenIdx].ctx
}
```

---

## 四层 priority 的协同工作流

将四层 priority 放在一起，完整的变异工作流如下：

```
                    ┌──────────────────────────────┐
                    │   corpus 中有 N 个程序        │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  第二层：chooseProgram()      │
                    │  按 signal 数量加权随机选择    │
                    │  （或按 Focus Area 选区域）   │
                    └──────────┬───────────────────┘
                               │ 选中程序 P
                    ┌──────────▼───────────────────┐
                    │  选择变异操作类型              │
                    │  Splice(43%) > Insert(22%)    │
                    │  > MutateArg(22%) > ...       │
                    └──────────┬───────────────────┘
                               │ 若 MutateArg
                    ┌──────────▼───────────────────┐
                    │  第四层：chooseCall()         │
                    │  按参数复杂度加权选择 call     │
                    └──────────┬───────────────────┘
                               │ 选中 call C
                    ┌──────────▼───────────────────┐
                    │  第四层：chooseArg()          │
                    │  按参数类型优先级选择 arg      │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  若 Insert：第三层 choose()   │
                    │  按优先级矩阵选下一个 syscall │
                    │  5% 完全随机逃逸              │
                    └──────────┬───────────────────┘
                               │ 生成/变异后代 P'
                    ┌──────────▼───────────────────┐
                    │  执行 P'，收集覆盖信息        │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  第一层：signalPrio()         │
                    │  计算每个 call 的信号优先级    │
                    │  DiffRaw → 只保留"更好的"     │
                    └──────────┬───────────────────┘
                               │ 有新 signal？
                    ┌──────────▼───────────────────┐
                    │  第二层：saveProgram()        │
                    │  P' 进入 corpus，权重 = signal│
                    │  正反馈闭环开始              │
                    └──────────────────────────────┘
```

---

## 拓展性分析

### 可能存在的问题

**1. Focus Area 的"冷启动"问题**

当 Focus Area 配置了特定的 CoverPCs，但 corpus 中尚未有程序覆盖这些 PC 时，该区域为空（`len(area.progs) == 0`），会被跳过。这意味着 Focus Area 在初始阶段完全无效，直到有程序碰巧覆盖了目标区域。对于大型内核模块，这可能需要很长时间。

一个改进方向是：在 Focus Area 为空时，退化为按全局 corpus 选择，而非直接跳过该区域。

**2. ChoiceTable 重建的开销**

当前 ChoiceTable 每 33（小 corpus）或 333（大 corpus）个程序重建一次，`BuildChoiceTable` 的时间复杂度为 O(N²)，其中 N 是 syscall 数量。对于 Linux（约 400 个 syscall），一次重建需要遍历 160,000 个元素对。虽然通过 goroutine 异步执行不阻塞主循环，但当 corpus 增长非常快时（如多个 fuzzer 并行运行），可能仍会成为瓶颈。

**3. 动态优先级的简单加法合并**

静态优先级和动态优先级通过简单的加法合并。但两者的量纲和含义不同——静态优先级基于类型分析（理论价值），动态优先级基于共现统计（经验价值）。简单相加可能让静态分析的价值被统计噪声淹没，尤其是在 corpus 较小、统计不可靠的早期阶段。

引入加权系数或根据 corpus 大小动态调整静态/动态比例，可能是一个值得探索的方向。

**4. 5% 随机逃逸的固定比例**

5% 的完全随机选择是一个经验值，注释中也承认"There were no deep ideas nor any calculations behind these numbers"。不同的内核子系统或 fuzzing 阶段可能需要不同的探索/利用比例——例如在 fuzzing 早期需要更多探索（更高的随机比例），后期需要更多利用（更低的随机比例）。

### 科研与工程价值

**从科研角度看**，syzkaller 的四层 priority 体系是一个经典的**层次化反馈系统**。底层（signal）提供原始信号，逐层抽象为程序选择、syscall 选择和参数选择策略。这种设计在强化学习（RL）中也有对应——signal 类似 reward，各层 priority 类似 policy，整个系统形成了一个无参数的在线学习过程。有趣的是，syzkaller 并没有使用 RL 或其他"高级"算法，而是通过精心设计的启发式规则达到了类似的效果。这引发了一个值得研究的问题：**在模糊测试中，简单但领域知识丰富的启发式，是否比通用的学习算法更有效？**

**从工程角度看**，四层 priority 的解耦设计值得借鉴。每一层都有明确的输入、输出和职责，修改某一层的策略不会影响其他层。例如，可以独立替换 ChoiceTable 的优先级计算算法（如引入 ML 模型），而不影响 signalPrio 或 corpus 选择逻辑。这种模块化设计使得 syzkaller 能够持续演进，而不必推翻整个架构。

---

## 设计哲学总结

回顾四层 priority，我们可以提炼出 syzkaller 的几个核心设计原则：

1. **"成功调用比失败调用更有价值"**——`signalPrio` 中 `errno == 0` 获得 +2 的权重加分。
2. **"覆盖越广的程序越值得深挖"**——corpus 优先级直接等于 signal 数量，形成正反馈闭环。
3. **"生产者 → 消费者优先"**——ChoiceTable 静态优先级通过 `w0.inout * w1.in * 3/2` 偏向资源流动方向。
4. **"高频共现对值得优先尝试"**——动态优先级通过 `sqrt()` 压缩计数，强调"是否共现"而非"共现多少次"。
5. **"5% 随机保证不陷入局部最优"**——ChoiceTable 的 choose 方法始终有 5% 概率完全随机选择。
6. **"复杂类型优先变异，常量不变异"**——`getMutationPrio` 为有 generator 的 Struct、多选项 Union、BufferCompressed 赋予最高优先级 10，而 ConstType 和 CsumType 为 0。

这些原则的共同目标是：在有限的执行预算内，最大化发现新覆盖的概率。

## 参考

- [pkg/signal/signal.go](https://github.com/google/syzkaller/blob/master/pkg/signal/signal.go) — Signal 数据结构与 DiffRaw 算法
- [pkg/fuzzer/fuzzer.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/fuzzer.go) — signalPrio 与 ChoiceTable 动态更新
- [pkg/fuzzer/cover.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/cover.go) — Cover 信号收集
- [pkg/corpus/prio.go](https://github.com/google/syzkaller/blob/master/pkg/corpus/prio.go) — ProgramsList 加权选择与 Focus Area
- [pkg/corpus/corpus.go](https://github.com/google/syzkaller/blob/master/pkg/corpus/corpus.go) — Corpus 去重与合并
- [prog/prio.go](https://github.com/google/syzkaller/blob/master/prog/prio.go) — ChoiceTable 优先级矩阵
- [prog/mutation.go](https://github.com/google/syzkaller/blob/master/prog/mutation.go) — 变异操作权重与 getMutationPrio
