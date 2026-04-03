---
title: "Syzkaller 的队列机制：从源码看内核模糊测试的调度哲学"
description: 深入分析 syzkaller 内核模糊测试框架中的多级队列架构，包括调度优先级、Triage 机制、deflake 算法与 Corpus 正反馈闭环
slug: syz-queue
date: 2026-04-03 00:00:00+0000
categories:
    - Fuzz
tags:
    - fuzz
    - syzkaller
    - kernel
weight: 1
---

# Syzkaller 的队列机制：从源码看内核模糊测试的调度哲学

文章使用**WorkBuddy**撰写。

## 1、引言：为什么 syzkaller 需要精心设计队列？

内核模糊测试面临一个天然的矛盾：**效率与探索广度的取舍**。

一方面，对于已知能触发新覆盖的程序，我们希望尽快完成 triage（分流）、minimize（最小化）和 smash（强化变异），让这些"有价值的程序"尽快固化为 corpus（语料库）的一部分；另一方面，如果所有资源都被这些"热点程序"占用，fuzzer 将停止探索新的代码路径，陷入局部最优。

syzkaller 通过一套**五级优先级队列系统**来平衡这一矛盾。不同来源、不同目的的执行请求被分配到不同优先级的队列中，调度器按序轮询，确保"紧急且重要的任务"不被忽视，同时又不会完全饿死"探索性任务"。

本文将从源码出发，逐层拆解这套调度机制。

---

## 2、总体架构：execQueues 结构体

所有队列逻辑的入口是 `fuzzer.go` 中的 `execQueues` 结构体：

```go
type execQueues struct {
    triageCandidateQueue *queue.DynamicOrderer
    candidateQueue       *queue.PlainQueue
    triageQueue          *queue.DynamicOrderer
    smashQueue           *queue.PlainQueue
    source               queue.Source
}
```

这里定义了四条具名队列和一个统一的 `source`。`source` 是真正对外暴露的调度入口——外部调用 `fuzzer.Next()` 时，实际上是从这个 `source` 拉取下一个执行请求。

`source` 在 `newExecQueues` 中被构建：

```go
func newExecQueues(fuzzer *Fuzzer) execQueues {
    ret := execQueues{
        triageCandidateQueue: queue.DynamicOrder(),
        candidateQueue:       queue.Plain(),
        triageQueue:          queue.DynamicOrder(),
        smashQueue:           queue.Plain(),
    }
    // Alternate smash jobs with exec/fuzz to spread attention to the wider area.
    skipQueue := 3
    if fuzzer.Config.PatchTest {
        skipQueue = 2
    }
    // Sources are listed in the order, in which they will be polled.
    ret.source = queue.Order(
        ret.triageCandidateQueue,
        ret.candidateQueue,
        ret.triageQueue,
        queue.Alternate(ret.smashQueue, skipQueue),
        queue.Callback(fuzzer.genFuzz),
    )
    return ret
}
```

五个 Source 被串联进 `queue.Order(...)` 中，形成一条**严格的优先级调度链**。

---

## 3、五级调度链：优先级从高到低

`queue.Order` 的实现非常简洁——每次调用 `Next()` 时，依次尝试每个 Source，遇到第一个非 nil 的结果立即返回：

```go
func (o *orderImpl) Next() *Request {
    for _, s := range o.sources {
        req := s.Next()
        if req != nil {
            return req
        }
    }
    return nil
}
```

这意味着**前面的队列永远优先于后面的队列**。以下是五个优先级层级的语义解释：

```
优先级 1 (最高): triageCandidateQueue  ← 候选程序的 triage 任务
优先级 2:        candidateQueue        ← 待验证的候选程序（来自 corpus 或 hub）
优先级 3:        triageQueue           ← 普通 fuzz 程序的 triage 任务
优先级 4:        smashQueue (Alternate) ← 对新 corpus 程序的强化变异（有节流）
优先级 5 (最低): genFuzz (Callback)    ← 生成/变异程序（常规 fuzz 循环）
```

**设计意图解析：**

- **候选程序优先（P1/P2）**：syzkaller 启动时会从本地 corpus 和 syz-hub 同步大量程序作为候选，这些程序是"已知有价值的种子"，需要尽快验证是否带来新 signal。在候选 triage 完成（`CandidateTriageFinished()`）之前，fuzzer 基本不会切换到纯探索模式。
- **triage 任务比 smash 优先（P3 > P4）**：发现新 signal 后立即进入 triage 流程，这是扩大 corpus 的核心路径，应该比"深挖已有 corpus"更紧迫。
- **smash 有节流（P4 with Alternate）**：smash 是对已入 corpus 的程序做大量变异，是"深度挖掘"。如果让它完全占用资源，就无法探索新的代码路径。`Alternate(smashQueue, 3)` 使得每 3 次 `Next()` 调用中，第 3 次会被强制跳过（返回 nil），把机会让给后面的 `genFuzz`。
- **genFuzz 兜底（P5）**：当所有队列都空时，始终可以生成/变异程序继续模糊测试，保证 fuzzer 不会闲置。

### Alternate 节流机制

```go
type alternate struct {
    base Source
    nth  int
    seq  atomic.Int64
}

func (a *alternate) Next() *Request {
    if a.seq.Add(1)%int64(a.nth) == 0 {
        return nil
    }
    return a.base.Next()
}
```

`Alternate` 用原子计数器 `seq` 跟踪调用次数，每 `nth` 次强制返回 nil。当 `nth=3` 时，smash 队列每 3 次调用中有 1 次会被跳过，约占 33% 的"让步率"。在 PatchTest 模式下，`nth=2` 让步率提升到 50%——因为 patch 测试更关注对现有 corpus 的全面变异，而非扩大覆盖。

---

## 4、底层队列实现：PlainQueue 与 DynamicOrderer

### 4.1 PlainQueue：简单高效的 FIFO 队列

`PlainQueue` 是标准的先进先出队列，用于 `candidateQueue` 和 `smashQueue`：

```go
type PlainQueue struct {
    mu    sync.Mutex
    queue []*Request
    pos   int
}

func (pq *PlainQueue) Submit(req *Request) {
    pq.mu.Lock()
    defer pq.mu.Unlock()
    // 队列长度超过 128 且已消费超过一半时进行压缩
    const minSizeToCompact = 128
    if pq.pos > len(pq.queue)/2 && len(pq.queue) >= minSizeToCompact {
        copy(pq.queue, pq.queue[pq.pos:])
        // ... 截断
    }
    pq.queue = append(pq.queue, req)
}
```

值得注意的是它的内存管理：并非每次 Pop 都缩容，而是在"已消费量超过一半且总量超过 128"时才做一次 copy+截断，避免频繁的内存分配。

### 4.2 DynamicOrderer：基于最小堆的动态优先级队列

`DynamicOrderer` 用于 `triageCandidateQueue` 和 `triageQueue`，它的特殊之处在于**支持在运行时动态创建子队列**，并严格保证先创建的子队列中的元素有更高的优先级：

```go
type DynamicOrderer struct {
    mu       sync.Mutex
    currPrio int
    ops      *priorityQueueOps[*Request]
}

func (do *DynamicOrderer) Append() Executor {
    do.mu.Lock()
    defer do.mu.Unlock()
    do.currPrio++
    return &dynamicOrdererItem{
        parent: do,
        prio:   do.currPrio,
    }
}

func (do *DynamicOrderer) Next() *Request {
    do.mu.Lock()
    defer do.mu.Unlock()
    return do.ops.Pop()
}
```

每次调用 `Append()` 都会分配一个递增的 `prio` 值。底层使用**最小堆**实现：

```go
func (pq priorityQueueImpl[T]) Less(i, j int) bool {
    // 堆顶是优先级数值最小的元素（即最先创建的子队列）
    return pq[i].prio < pq[j].prio
}
```

**为什么 triage 队列需要动态排序？**

triage 任务是随着 fuzz 执行动态产生的——每发现一批新 signal，就会创建一个新的 triageJob，调用 `triageQueue.Append()` 获得一个专属子执行器。由于 triage 任务本身又需要多次执行（deflake 需要多次运行），这些执行请求通过各自的子执行器提交，`DynamicOrderer` 保证了**先创建的 triage job 的请求永远排在后创建的 job 之前**，形成了 triage 任务内部的 FIFO 秩序，避免新涌入的 triage 任务"插队"导致旧任务饿死。

---

## 5、候选程序（Candidate）的生命周期

候选程序是 syzkaller 重要的"种子输入"来源，主要来自本地持久化 corpus 和 syz-hub 的同步。以下是其完整生命周期：

### 5.1 加入队列

```go
func (fuzzer *Fuzzer) AddCandidates(candidates []Candidate) {
    fuzzer.statCandidates.Add(len(candidates))
    for _, candidate := range candidates {
        req := &queue.Request{
            Prog:      candidate.Prog,
            ExecOpts:  setFlags(flatrpc.ExecFlagCollectSignal),
            Stat:      fuzzer.statExecCandidate,
            Important: true,   // 候选程序被标记为重要，支持 crash 后重试
        }
        fuzzer.enqueue(fuzzer.candidateQueue, req, candidate.Flags|progCandidate, 0)
    }
}
```

候选程序携带 `Important: true` 标志，这对于 Retry 机制有重要意义（详见第 8 节）。

### 5.2 执行与结果处理

每个请求执行完成后，都会触发 `processResult`：

```go
func (fuzzer *Fuzzer) processResult(req *queue.Request, res *queue.Result, flags ProgFlags, attempt int) bool {
    dontTriage := flags&progInTriage > 0 || res.Status == queue.Hanged
    var triage map[int]*triageCall
    if req.ExecOpts.ExecFlags&flatrpc.ExecFlagCollectSignal > 0 && res.Info != nil && !dontTriage {
        for call, info := range res.Info.Calls {
            fuzzer.triageProgCall(req.Prog, info, call, &triage)
        }
        // ...
        if len(triage) != 0 {
            queue, stat := fuzzer.triageQueue, fuzzer.statJobsTriage
            if flags&progCandidate > 0 {
                // 候选程序触发的 triage 进入更高优先级的 triageCandidateQueue
                queue, stat = fuzzer.triageCandidateQueue, fuzzer.statJobsTriageCandidate
            }
            job := &triageJob{ /* ... */ }
            fuzzer.startJob(stat, job)
        }
    }
    // ...
}
```

这里有一个关键设计：**候选程序触发的 triage 任务会进入 `triageCandidateQueue`（优先级 1）**，而普通 fuzz 程序触发的 triage 进入 `triageQueue`（优先级 3）。这保证了候选 triage 始终优先于普通 triage。

### 5.3 重试机制

```go
// Corpus candidates may have flaky coverage, so we give them a second chance.
maxCandidateAttempts := 3
if req.Risky() {
    maxCandidateAttempts = 2
    if fuzzer.Config.Snapshot || res.Status == queue.Hanged {
        maxCandidateAttempts = 0
    }
}
if len(triage) == 0 && flags&ProgFromCorpus != 0 && attempt < maxCandidateAttempts {
    fuzzer.enqueue(fuzzer.candidateQueue, req, flags, attempt+1)
    return false
}
```

来自 corpus 的候选程序如果没有触发任何新 signal，会被**最多重试 3 次**——因为 corpus 程序经过历史筛选，应当能持续触发 signal，若一次失败很可能是 flaky。如果程序标记为 Risky（曾经导致 crash），最多重试 2 次；在 Snapshot 模式下不重试（因为 Snapshot 模式确定性强，一次失败即代表不复现）。

---

## 6、Triage 机制：从 signal 到 corpus 的完整路径

triage 是 syzkaller 确认并固化新发现的核心流程，也是队列系统与 corpus 增长之间的桥梁。

### 6.1 触发条件

```go
func (fuzzer *Fuzzer) triageProgCall(p *prog.Prog, info *flatrpc.CallInfo, call int, triage *map[int]*triageCall) {
    if info == nil {
        return
    }
    prio := signalPrio(p, info, call)
    newMaxSignal := fuzzer.Cover.addRawMaxSignal(info.Signal, prio)
    if newMaxSignal.Empty() {
        return  // 没有新的最大 signal，不触发 triage
    }
    // ...
    (*triage)[call] = &triageCall{
        errno:     info.Error,
        newSignal: newMaxSignal,
        signals:   [deflakeNeedRuns]signal.Signal{signal.FromRaw(info.Signal, prio)},
    }
}
```

`addRawMaxSignal` 尝试将当前 signal 合并进全局最大 signal 集合，只有当出现**此前从未见过的 signal 位**时才返回非空集合，触发 triage。

### 6.2 deflake：确认 signal 的稳定性

triage 最核心的子步骤是 deflake——通过多次重复执行来过滤掉 flaky（不稳定）的 signal：

```go
const (
    deflakeNeedRuns         = 3   // 普通 fuzz 程序：3/5 次出现即视为稳定
    deflakeMaxRuns          = 5
    deflakeNeedCorpusRuns   = 2   // corpus 程序：2/6 次出现即视为稳定
    deflakeMinCorpusRuns    = 4
    deflakeMaxCorpusRuns    = 6
    deflakeTotalCorpusRuns  = 20  // corpus 程序最多执行 20 次
    deflakeNeedSnapshotRuns = 2   // Snapshot 模式：2 次即可
)
```

**为什么 corpus 程序和 fuzz 程序的 deflake 参数不同？**

- **普通 fuzz 程序（3/5）**：这类程序是新发现的，需要严格确认其 signal 的真实性，避免把 flaky 的假阳性结果固化到 corpus 中，引发后续大量无效的 minimize/smash/hints 工作。
- **corpus 程序（2/6）**：corpus 程序在最初加入时已经通过了严格的 3/5 检验。syzkaller 重启后会对 corpus 进行 retriage，此时放宽标准（2/6）有两个好处：其一，减少无谓的执行次数，更快完成 retriage；其二，降低因内核版本变化导致程序被误判"不再有效"的概率。

deflake 还有一个工程细节：每次重执行时会通过 `Avoid` 字段标记已用过的 VM，尽量让同一个程序在**不同 VM** 上执行，降低因某台 VM 状态异常导致的误判。

### 6.3 minimize → smash/hints → corpus.Save

deflake 通过后，triage 会进入 minimize 阶段，将程序精简到能触发 signal 的最小形式，然后：

```go
func (job *triageJob) handleCall(call int, info *triageCall) {
    // ...
    // 启动 smash job
    job.fuzzer.startJob(job.fuzzer.statJobsSmash, &smashJob{
        exec: job.fuzzer.smashQueue,
        p:    p.Clone(),
    })
    // 启动 hints job（KCOV comparison 引导的变异）
    if job.fuzzer.Config.Comparisons && call >= 0 {
        job.fuzzer.startJob(job.fuzzer.statJobsHints, &hintsJob{
            exec: job.fuzzer.smashQueue,
            // ...
        })
    }
    // 保存到 corpus
    job.fuzzer.Config.Corpus.Save(input)
}
```

注意：smashJob 和 hintsJob 都将 `smashQueue` 作为执行器，它们提交的请求会进入 `smashQueue`（优先级 4），而不是直接参与 triage 竞争。这样形成了一个清晰的流水线：

```
新 signal 发现
      ↓
triageJob (triage 队列，P1/P3)
      ↓ deflake + minimize
smashJob / hintsJob (smash 队列，P4)
      ↓ 提交变异程序执行
corpus.Save (固化)
      ↓
ChoiceTable 更新 → 反馈给 genFuzz
```

---

## 7、smash 与 hints：深度挖掘已知 corpus

### 7.1 smashJob：暴力变异

`smashJob` 对刚入 corpus 的程序做 25 次随机变异并执行，目的是在新发现的代码路径附近进行密集探索：

```go
func (job *smashJob) run(fuzzer *Fuzzer) {
    const iters = 25
    rnd := fuzzer.rand()
    for i := 0; i < iters; i++ {
        p := job.p.Clone()
        p.Mutate(rnd, prog.RecommendedCalls, fuzzer.ChoiceTable(),
            fuzzer.Config.NoMutateCalls, fuzzer.Config.Corpus.Programs())
        result := fuzzer.execute(job.exec, &queue.Request{
            Prog:     p,
            ExecOpts: setFlags(flatrpc.ExecFlagCollectSignal),
            Stat:     fuzzer.statExecSmash,
        })
        if result.Stop() {
            return
        }
    }
}
```

### 7.2 hintsJob：比较引导的精准变异

`hintsJob` 利用 KCOV 的 comparison tracing（kcov_cmp）功能，收集程序执行时内核中的比较操作数，然后将程序参数替换为这些操作数进行变异。这是一种**数据流引导的变异策略**，能够精准突破条件检查：

```go
func (job *hintsJob) run(fuzzer *Fuzzer) {
    // 执行 3 次，取 comparison 数据的交集（过滤 flaky 值）
    for i := 0; i < 3; i++ {
        result := fuzzer.execute(job.exec, &queue.Request{
            ExecOpts: setFlags(flatrpc.ExecFlagCollectComps),
            // ...
        })
        // 取交集
        comps.InplaceIntersect(got)
    }
    // 用稳定的 comparison 值驱动变异
    p.MutateWithHints(job.call, comps, func(p *prog.Prog) bool {
        result := fuzzer.execute(job.exec, &queue.Request{
            ExecOpts: setFlags(flatrpc.ExecFlagCollectSignal),
        })
        return !result.Stop()
    })
}
```

hints 同样提交到 `smashQueue`，与 smash 共享优先级 4，在 `Alternate` 节流下与 `genFuzz` 交替执行。

---

## 8、Retry 机制：对抗 VM 不稳定性

内核 fuzzing 的特殊性在于：VM 随时可能因为被测程序而崩溃或重启。`queue.Retry` 专门处理这种情况：

```go
func (r *retryer) done(req *Request, res *Result) bool {
    switch res.Status {
    case Success, ExecFailure, Hanged:
        return true   // 正常完成，不重试
    case Restarted:
        // VM 重启了，但请求还没完成——无条件重新入队
        r.pq.Submit(req)
        return false
    case Crashed:
        // VM crash 了——只对标记为 Important 的请求重试一次
        if req.Important && !req.onceCrashed {
            req.onceCrashed = true
            r.pq.Submit(req)
            return false
        }
        return true
    }
}
```

**Restarted vs Crashed 的不同处理策略**很有意思：

- `Restarted`（VM 正常重启，如周期性 reset）：**无条件重试**。因为 VM 重启本身与程序无关，程序完全有可能在下一个 VM 上正常执行。
- `Crashed`（VM 被程序搞崩了）：**仅对 Important 请求重试一次**。因为可能是程序本身触发了内核 bug，重试一次是为了确认是否是 flaky crash，但不能无限重试，否则一个持续触发 crash 的程序会垄断执行资源。

`candidateQueue` 中的程序都带有 `Important: true`，因此候选程序在 crash 后会获得一次重试机会。triage 执行（triageJob 中的 execute）也会将请求标记为 `Important: true`（见 `job.execute` 函数）。

---

## 9、Distributor：基于 VM 回避的调度优化

`Distributor` 是一个可选的调度增强层，它在 `Next()` 中接受 VM ID 参数，并实现了一种**软 VM 回避**机制：

```go
func (dist *Distributor) Next(vm int) *Request {
    dist.noteActive(vm)
    if req := dist.delayed(vm); req != nil {
        return req
    }
    for {
        req := dist.source.Next()
        // 如果请求希望回避此 VM，且还有其他活跃 VM 可用，则延迟此请求
        if req == nil || !contains(req.Avoid, vm) || !dist.hasOtherActive(req.Avoid) {
            return req
        }
        dist.delay(req)
    }
}
```

`Avoid` 字段在 triage 的 deflake 阶段被设置：每次 deflake 执行后，会将用过的 VM 加入 `Avoid` 列表，希望下一次执行在不同的 VM 上进行。

**为什么需要 VM 回避？**

内核 fuzzing 中，VM 的状态可能被之前的程序污染（如残留的内核对象、竞争条件下的状态不一致）。如果 deflake 的多次执行都在同一台 VM 上进行，flaky signal 的来源可能是 VM 状态污染而非程序本身的不确定性。通过 VM 回避，每次 deflake 执行在不同的"干净"VM 上运行，能更准确地判断 signal 的稳定性。

回避是**软约束**：如果延迟队列中的请求等待超过 1000 个调度序列号（即其他 VM 已经不活跃了），请求会被强制分配出去，避免永远等待。

---

## 10、Deduplicator：避免重复执行

`Deduplicator` 是另一个调度增强层，维护了一个已运行请求的哈希表：

```go
func (d *Deduplicator) Next() *Request {
    for {
        req := d.source.Next()
        hash := req.hash()
        d.mu.Lock()
        entry, ok := d.mm[hash]
        if !ok {
            d.mm[hash] = &duplicateState{}  // 首次见到此请求，正常执行
        } else if entry.res == nil {
            entry.queued = append(entry.queued, req)  // 正在执行中，排队等结果
        } else {
            req.Done(entry.res.clone())  // 已有结果，直接复用
        }
        d.mu.Unlock()
        if !ok {
            req.OnDone(d.onDone)
            return req
        }
    }
}
```

当相同的程序（以 `Prog.Serialize()` + `ExecOpts` 的哈希标识）被多次提交时，`Deduplicator` 会将后来的请求挂起，等第一个执行完成后广播结果给所有等待者，避免重复执行同样的程序。

---

## 11、Corpus 对队列的反馈：正反馈闭环

corpus 的增长反过来影响 `genFuzz` 阶段的程序生成和变异。`genFuzz` 中的变异路径调用：

```go
func mutateProgRequest(fuzzer *Fuzzer, rnd *rand.Rand) *queue.Request {
    p := fuzzer.Config.Corpus.ChooseProgram(rnd)
    // ...
}
```

`ChooseProgram` 并非简单随机选择，而是**按程序触发的 signal 数量进行加权**（权重越高代表该程序覆盖更多路径）。corpus 中 signal 丰富的程序被选中变异的概率更高，形成了一个正反馈循环：

```
发现新 signal
    ↓
triage → minimize → 加入 corpus（signal 权重高）
    ↓
被 ChooseProgram 优先选中 → 更多变异
    ↓
更可能发现更多新 signal（在相邻代码路径上）
    ↓
再次触发 triage ...
```

同时，`ChoiceTable` 会随着 corpus 增长而更新，记录各个 syscall 的使用频率和有效参数，指导程序生成时的系统调用选择，使新生成的程序更倾向于复现和拓展已发现的代码路径。

---

## 12、设计哲学总结

| 设计决策 | 工程考量 |
|----------|----------|
| 五级严格优先级 | 保证"紧急任务"（triage）不被"常规任务"（genFuzz）饿死，同时 genFuzz 永远有机会执行（兜底） |
| Alternate 节流 smash | 防止深度挖掘占用全部资源，强制保留探索广度 |
| DynamicOrderer | 允许 triage job 在运行时动态分配子队列，同时保证先创建的 job 先完成（公平性） |
| corpus 程序 deflake 放宽（2/6 vs 3/5） | 加速 retriage，减少因 flaky 程序被误删导致的重复工作 |
| VM 回避（Distributor） | 减少 VM 状态污染对 deflake 准确性的干扰 |
| Retry 区分 Restarted/Crashed | Restarted 无损重试，Crashed 限次重试，避免"毒药程序"垄断资源 |
| Important 标记 | 精确控制哪些请求值得在 crash 后重试，节约执行资源 |
| 加权 corpus 选择 | 形成正反馈，引导变异朝已知有价值的代码路径聚焦 |

syzkaller 的队列机制充分体现了内核 fuzzing 的工程取舍艺术：**既要快速跟进新发现（高优先级的 triage 链），又要持续探索未知（genFuzz 的最低优先级保底）；既要深挖已知路径（smash/hints），又不能让深挖垄断资源（Alternate 节流）**。这套机制在多年的生产实践中不断演化，是 syzkaller 能持续发现内核漏洞的重要工程基础。

---

## 参考

- [syzkaller source: pkg/fuzzer/fuzzer.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/fuzzer.go)
- [syzkaller source: pkg/fuzzer/queue/queue.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/queue/queue.go)
- [syzkaller source: pkg/fuzzer/queue/retry.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/queue/retry.go)
- [syzkaller source: pkg/fuzzer/queue/distributor.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/queue/distributor.go)
- [syzkaller source: pkg/fuzzer/job.go](https://github.com/google/syzkaller/blob/master/pkg/fuzzer/job.go)
