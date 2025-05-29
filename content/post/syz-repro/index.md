---
title: Syz-repro
description: syz-repro的一些分析
slug: syz-repro
date: 2025-05-29 00:00:00+0000
categories:
    - Fuzz
tags:
    - fuzz
    - syzkaller
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# Syz-Repro分析


[syzkaller/tools/syz-repro/repro.go at master · google/syzkaller](https://github.com/google/syzkaller/blob/master/tools/syz-repro/repro.go)

## syz-repro的使用

```
./bin/syz-repro -config=my.cfg /path/to/crash/log
```

使用syz-repro对log中包含的prog进行复现，这个流程包括

1、提取出可以触发crash的初始prog

2、对这个prog进行简化，基本采用的是逐个syscall削减简化

3、对prog的参数等进行简化

4、将prog转化为C poc

5、对C poc进行简化





## 程序最小化

pkg/repro/repro.go Run runInner reproCtx.run()

minimizeProg对extract出来的程序进行minimize，在这之中调用prog.Minimize进行简化

```jsx
// Minimize calls and arguments.
func (ctx *reproContext) minimizeProg(res *Result) (*Result, error) {
	ctx.reproLogf(2, "minimizing guilty program")
	start := time.Now()
	defer func() {
		ctx.stats.MinimizeProgTime = time.Since(start)
	}()

	mode := prog.MinimizeCrash
	if ctx.fast {
		mode = prog.MinimizeCallsOnly
	}
	res.Prog, _ = prog.Minimize(res.Prog, -1, mode, func(p1 *prog.Prog, callIndex int) bool {
		if len(p1.Calls) == 0 {
			// We do want to keep at least one call, otherwise tools/syz-execprog
			// will immediately exit.
			return false
		}
		ret, err := ctx.testProg(p1, res.Duration, res.Opts, false)
		if err != nil {
			ctx.reproLogf(2, "minimization failed with %v", err)
			return false
		}
		return ret.Crashed
	})

	return res, nil
}
```

这里会调用prog中的Minimize函数对程序进行简化

包含的参数：

1、prog 程序列表

2、index 开始简化的syscall下标

3、mode 简化模式

4、pred 测试函数，这个函数会对简化后的程序进行测试，观察其能否触发Bug

prog minimization.go中

```jsx
Minimize函数中调用removeCalls进行简化，移除不必要的 syscall
p0, callIndex0 = removeCalls(p0, callIndex0, pred)

func removeCalls(p0 *Prog, callIndex0 int, pred minimizePred) (*Prog, int) {
	if callIndex0 >= 0 && callIndex0+2 < len(p0.Calls) {
		// It's frequently the case that all subsequent calls were not necessary.
		// Try to drop them all at once.
		p := p0.Clone()
		for i := len(p0.Calls) - 1; i > callIndex0; i-- {
			p.RemoveCall(i)
		}
		if pred(p, callIndex0, statMinRemoveCall, "trailing calls") {
			p0 = p
		}
	}

	if callIndex0 != -1 {
		p0, callIndex0 = removeUnrelatedCalls(p0, callIndex0, pred)
	}
	//这里可以看到是逐个syscall尝试进行移除的
	for i := len(p0.Calls) - 1; i >= 0; i-- {
		if i == callIndex0 {
			continue
		}
		callIndex := callIndex0
		if i < callIndex {
			callIndex--
		}
		p := p0.Clone()
		p.RemoveCall(i)
		if !pred(p, callIndex, statMinRemoveCall, fmt.Sprintf("call %v", i)) {
			continue
		}
		p0 = p
		callIndex0 = callIndex
	}
	return p0, callIndex0
}
```



## 对crash log的处理

log里面存放的内容：

1、minimize之前的prog

2、minimize这个过程的prog，以及是否crashed

3、C程序的生成以及minimize

minimize的方式

prog/minimization.go Minimize对程序进行minimize处理

处理log的方式

prog/parse.go ParseLog对log文件进行处理

```jsx
函数 `ParseLog` 处理日志的方式如下：
1. 初始化一个 `LogEntry` 结构体和一些变量。
2. 通过循环逐行读取日志数据。
3. 如果行中包含 "executing program "，则解析并创建一个新的 `LogEntry`。
4. 将当前行追加到 `cur` 中，并尝试反序列化为程序。
5. 如果反序列化成功并且存在故障调用，则设置相应的故障属性。
6. 最后，将所有解析的 `LogEntry` 返回。
```



## Log处理

可以只在log里面放一个最原始的未简化的prog

后面程序会先尝试复现这个prog，并且会记录所有相关的时间

实验测试结果



## 结果保存

复现的结果保存到相关的文件里。

```
		fmt.Printf("opts: %+v crepro: %v\n\n", res.Opts, res.CRepro)
		//将程序序列化，写入相关文件
		progSerialized := res.Prog.Serialize()
		fmt.Printf("%s\n", progSerialized)
		if err = osutil.WriteFile(*flagOutput, progSerialized); err == nil {
			fmt.Printf("program saved to %s\n", *flagOutput)
		} else {
			log.Logf(0, "failed to write prog to file: %v", err)
		}

		if res.Report != nil && *flagTitle != "" {
			recordTitle(res, *flagTitle)
		}
		if res.CRepro {
			recordCRepro(res, *flagCRepro)
		}
		if *flagStrace != "" {
			result := repro.RunStrace(res, cfg, reporter, pool)
			recordStraceResult(result, *flagStrace)
		}
```

