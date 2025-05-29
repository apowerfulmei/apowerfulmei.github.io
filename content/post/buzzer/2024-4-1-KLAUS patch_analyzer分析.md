
[](https://github.com/wupco/KLAUS/tree/main/Syzpatch/patch_analyzer)

## KAMAIN

全局变量定义，同时也是参数定义，可以通过在命令后加上 “--debug-verbose=2”制定变量值

```
cl::opt<unsigned>
    VerboseLevel("debug-verbose",
                 cl::desc("Print information about actions taken"),
                 cl::init(0));
```

在analyze_patch.py文件中加入相关参数定义即可

```
            if res != 0:
                print("res not 0: "+str(res))
                print("cmd: "+cmd)
                cmd += " --disable-llvm-diff=1"
                os.system(cmd)
```



## 核心文件 ChangeAnalysis

### run

两种模式。

##### 1、LLVM未启用

```
    Function *raw = rawModule->getFunction(changedFunc->getName());
    Function *patched = patchedModule->getFunction(changedFunc->getName());
    doAnalysis(&raw->getEntryBlock(), &patched->getEntryBlock());
    Refine();
    storeRes();
```

##### 2、LLVM启用

首先使用llvm diff查找原始模块和补丁模块中不同的函数和basic block

随后调用doAnalysis对这些模块进行分析

genSetInsert



refine

storeRes: store result，将分析的结果保存到文件中



### doAnalysis

```
findRange
findVariables
analyzeValueSetRange
compareValues
```



##### 可以关注的一些输出点



传入参数 bb.first与bb.second

##### 1.调用findRange对参数进行分析，得到给定基本块范围

findRange对传入的参数进行基本块分析，返回目标的直接支配块和直接后支配块

```
  // assert(DTBB && PDTBB);
  // KA_LOGS(0, "Found dominator at " << *DTBB);
  // KA_LOGS(0, "Found postdominator at " << *PDTBB);
··// dominator tree basic block 
  // post dominator tree basic block
  return std::make_pair(DTBB, PDTBB); 
```

##### 2.调用findVariables查找感兴趣的变量

findVariables将查找目标函数中涉及到的感兴趣的变量

其中涉及到别名分析（alias analysis），是[编译器理论](https://zh.wikipedia.org/wiki/编译器)中的一种程序分析技术。当程序中同时出现两个甚至多个符号代表同样一个内存位置时，这些符号便可称作**别名**。与此相对应的，当两个或更多指针指向同一个地址时，那些指针称作[别名指针](https://zh.wikipedia.org/wiki/别名_(计算))。别名分析则是判断一个程序内是否存在别名的算法。

收集函数的所有参数，将其放到resVars中，收集其中为指针类型的参数，放到vars中

```
  std::set<Value *> vars;
  for (auto &arg : func->args()) {
    // all args are interesting
    resVars.insert(&arg);
    if (arg.getType()->isPointerTy())
      vars.insert(&arg);
  }
```

收集指令类型为指针类型的指令，将其添加到vars中

```
  for (Instruction &I : instructions(*func)) {
    if (I.getType()->isPointerTy())
      vars.insert(&I);
  }
```

找到所有调试信息，将其相关遍历添加到resVars中

```
    if (auto *dbgV = dyn_cast<DbgValueInst>(I)) {
      dbgValueVec.push_back(dbgV);
      dbgVars.insert(dbgV->getVariableLocation());
      // all debug variables are interesting.
      resVars.insert(dbgV->getVariableLocation());
```

找到类型为**GetElementPtr**的指令，将其添加到resVars中

在编写 LLVM IR 代码时，`GetElementPtrInst` （GEP）指令通常用于计算指针的偏移量，以便访问数组元素、结构体字段或者复合类型的成员。因此，GEP 指令本身并不是被限制在特定的使用情况下，而是可以在各种上下文中使用。

然而，这段代码中将 GEP 指令视为“合法”使用的前提条件是其使用者为 `CallInst`、`LoadInst` 或 `StoreInst`，这是因为在实际的程序中，GEP 指令通常用于计算指针的地址，而这些指令则是对内存进行读取或写入操作的典型场景。具体来说：

- `CallInst`：通常表示函数调用，可能会将 GEP 指令计算的地址传递给函数的参数，用于访问函数内部的数据结构或者传递参数。
- `LoadInst`：表示从内存中加载数据到寄存器或内存中的其他位置，GEP 指令计算的地址可能用于指示要加载的数据的位置。
- `StoreInst`：表示将数据写入内存的操作，GEP 指令计算的地址可能用于指示要写入数据的位置。

因此，当 GEP 指令的使用者为这些指令时，可以合理地假设 GEP 指令是用于地址计算的有效用法，这样的 GEP 指令可能是程序正确性的关键部分。而如果 GEP 指令被其他类型的指令使用，可能就需要更详细的分析和理解程序的逻辑，以确定其正确性。因此，这段代码将仅在 GEP 指令的使用者为 `CallInst`、`LoadInst` 或 `StoreInst` 时，将 GEP 指令视为“合法”使用。

```
    if (isa<GetElementPtrInst>(I)) {
      bool validGEI = false;
      for (auto *user_ : I->users()) {
        User *user = removeBCI(user_);
        //用于移除 BitCastInst 指令的链式结构，返回链中最终的非 BitCastInst 指令
        // check if it is valid
        //检查用户是否为CallInst、LoadInst或者StoreInst
        if (isa<CallInst>(user) || isa<LoadInst>(user) ||
            isa<StoreInst>(user)) {
          validGEI = true;
          break;
        }
      }

      if (validGEI)
        resVars.insert(I);
```

处理调用指令，将其操作数添加到resVars中

```
    if (isa<CallInst>(I)) {
      for (unsigned i = 0; i < I->getNumOperands(); i++) {
        resVars.insert(I->getOperand(i));
      }
    }
```

##### 3.调用analyzeValueSetRange，返回变量的可能值以及相应的约束

```
  GenSetPair rawPair = analyzeValueSetRange(raw, rawVars);
  GenSetPair patchedPair = analyzeValueSetRange(patched, patchedVars);
```

此函数得到的是前文中的block范围中涉及到的变量set的可能的值以及这些值对应的约束、条件信息

传入的参数中为前文分析得到的基本块范围以及感兴趣的变量



changeSet为一个集合，对changeSet中的每一个BB进行分析，查找其继任者，如果未包含到changeSet中，则将其放到changeSet中，**找到从起始节点到重点的所有路径**

```
  changeSet.push_back(start);
  while (changed) {
    changed = false;
    for (auto b : changeSet) {
      if (b == end)
        continue;
      for (auto *succ : successors(b)) {
        if (std::find(changeSet.begin(), changeSet.end(), succ) ==
            changeSet.end()) {
          changeSet.push_back(succ);
          changed = true;
        }
      }
    }
  }
```

​	对基本块进行数据流分析，初始化基本块的入口信息、出口信息、条件入口信息和条件出口信息

```
    // init block entry if it is not
    // 初始化基本块的入口和出口信息
    if (blockEntry.find(current) == blockEntry.end()) {
      GenSet blockUseInfo;
      GenSet blockDefInfo;
      blockEntry[current] = std::make_pair(blockDefInfo, blockUseInfo);
    }

    if (blockExit.find(current) == blockExit.end()) {
      GenSet blockUseInfo;
      GenSet blockDefInfo;
      blockExit[current] = std::make_pair(blockDefInfo, blockUseInfo);
    }
    //初始化条件入口信息和条件出口信息

    if (blockCondEntry.find(current) == blockCondEntry.end()) {
      std::set<Cond> condSet;
      Cond cond;
      // only the dominator worth init condition
      // otherwise it will generate a lot duplicated conditions
      if (current == start) {
        cond.push_back(std::make_pair(nullptr, 0));
        condSet.insert(cond);
      }
      blockCondEntry[current] = condSet;
    }

    if (blockCondExit.find(current) == blockCondExit.end()) {
      blockCondExit[current] = blockCondEntry[current];
    }
```

对于非入口块的每一个基本块进行分析，根据前驱基本块的出口信息更新当前基本块的条件入口信息。同时，进行条件合并操作，检查是否存在条件合并点，将重复的条件进行合并。



```
    if (current != start) {
      // condition set to reach current block
      blockCondEntry[current].clear();
      // 对于基本块的每一个前驱基本块
      for (auto *pre : predecessors(current)) {
        if (blockExit.find(pre) != blockExit.end()) {
        //当这些基本块的出口不为终点时
          // union the result
          auto preCondNode = getCond(std::make_pair(pre, current), vars);
          //getCond获取条件分支节点
          if (preCondNode.first && loggedCondNode.insert(preCondNode).second) {
            KA_LOGS(0, "adding condNode " << *preCondNode.first
                                          << preCondNode.second);
          }
          // could be a conditional jump
          // could not be a conditional jump

          // should check if this condNode is already in the entry
          auto preCondSet = blockCondExit[pre];
          std::set<Cond> newCurCondSet;

          // 1. sanitizer the preCondSet itself
          for (auto preCond : preCondSet) {
            if (preCond.size() == 0) {
              // precond is not initialized, skip this
              // otherwise we will see a lot ofduplicated
              // conditions
              continue;
            }
            // do a sanitization here, remove redudent cond
            Cond curCond(preCond);
            if (containCondNodeBranch(curCond, preCondNode) != -1) {
            //检查是否存在条件合并点，存在则将其合并
              // it is contradict, then it is a merge point
              // merge them!
              sanitizeCond(curCond, preCondNode);
            } else {
              // avoid redudent condNode
              if (!containCondNode(curCond, preCondNode)) {
              //如果不包含，则将其包含到其中
                curCond.push_back(preCondNode);
              }
            }

            newCurCondSet.insert(curCond);
          }

          // 2. sanitizer the preCondSet with each other
          // TODO
          for (auto cond : newCurCondSet) {
          }

          // KA_LOGS(0, "dumping conditions, size "<<newCurCondSet.size());
          // dump(newCurCondSet);

          // union new curCondSet
          unionCondSet(blockCondEntry[current], newCurCondSet);

          unionSet(blockEntry[current].first, blockExit[pre].first);
          unionSet(blockEntry[current].second, blockExit[pre].second);
        } else {
          // predecessor is not ready, skip this block
          // KA_WARNS(0, "predecessor is not initialized : " << *pre);
          continue;
        }
      }
```



这段代码主要用于基本块的数据流分析，通过遍历基本块的前驱基本块，根据前驱基本块的出口信息更新当前基本块的条件入口信息

这段代码实现了对给定基本块对（BasicBlockPair）的数据流分析，以确定在给定一组变量（ValueSet vars）的情况下，每个变量的可能值范围。



主要步骤如下：

1. 初始化数据结构和日志记录。
2. 使用宽度优先搜索（BFS）的方法遍历从起始基本块到结束基本块的所有基本块，并将它们添加到changeSet队列中。
3. 针对每个基本块，初始化基本块的入口和出口信息，以及条件入口和条件出口信息。
4. 对于除起始基本块外的其他基本块，根据前驱基本块的出口信息更新当前基本块的条件入口信息。同时，进行条件合并操作，检查是否存在条件合并点，将重复的条件进行合并。
5. 更新基本块的条件出口信息，并将其与入口信息进行合并。如果发生了变化，则将当前基本块的后继基本块添加到changeSet队列中。
6. 对于起始基本块，进行基本块内部的数据流分析，计算其入口和出口信息，并进行更新。
7. 循环直到changeSet队列为空，即所有基本块都已分析完毕。
8. 返回最终结果，即结束基本块的出口信息。

```
  assert(blockExit.find(end) != blockExit.end() && "Found nothing at the end");
  return blockExit[end];
  这个东西就是文章中提到的"S"
```

该算法通过分析每个基本块的入口和出口信息，以及条件入口和条件出口信息，从而确定了在给定一组变量情况下的可能值范围。





##### 4.调用compareValues进行对比

这段代码实现了对原始和修补版本的定义集合和使用集合进行比较，以找出newdef、deaddef、newuse。具体步骤如下：

1. （Dead Def）：
   - 遍历原始版本的定义集合（raw.first）中的每个元素（地址和条件值对）。
   - 检查该地址在修补版本的定义集合（patched.first）中是否存在：
     - 如果存在，则说明该地址在修补版本中仍然存在，因此不是死代码。进一步检查其条件值对：
       - 如果某个条件值对在原始版本中存在，但在修补版本中不存在，则将该条件值对添加到死代码集合中。
     - 如果不存在，则将该地址及其条件值对全部添加到死代码集合中。

2. （New Def）：
   - 遍历修补版本的定义集合（patched.first）中的每个元素。
   - 检查该地址在原始版本的定义集合中是否存在：
     - 如果存在，则说明该地址在原始版本中仍然存在，因此不是新的定义。进一步检查其条件值对：
       - 如果某个条件值对在修补版本中存在，但在原始版本中不存在，则将该条件值对添加到新的定义集合中。
     - 如果不存在，则将该地址及其条件值对全部添加到新的定义集合中。

3. （New Use）：
   - 遍历修补版本的使用集合（patched.second）中的每个元素。
   - 检查该地址在原始版本的使用集合中是否存在：
     - 如果存在，则说明该地址在原始版本中仍然存在，因此不是新的使用。进一步检查其条件值对：
       - 如果某个条件值对在修补版本中存在，但在原始版本中不存在，则将该条件值对添加到新的使用集合中。
     - 如果不存在，则将该地址及其条件值对全部添加到新的使用集合中。

4. 最后，打印出新的定义集合、死代码集合和新的使用集合。

### analyzeBasicBlock

这段代码实现了对基本块进行分析，找出其中的使用（Use）和定义（Def）信息，并根据给定的前置条件集合（preCondSet）更新使用信息（useInfo）和定义信息（defInfo）。具体步骤如下：

1. 遍历基本块中的每个指令。

2. 对于 LoadInst 指令，解析其操作数，获取其使用的值，并根据该值更新使用信息（useInfo）：
   - 如果该值不在使用信息中，则将其加入使用信息中，并使用给定的前置条件集合初始化条件值集合。
   
   - 如果该值已经在使用信息中，则检查每个条件值对，如果前置条件不在条件值对中，则添加前置条件到条件值对中，并标记发生了改变。
   
     
   
3. 对于 StoreInst 和 CallInst 指令，解析其操作数，获取存储的值和存储的地址，并根据这两个值更新定义信息（defInfo）：
   - 如果存储的地址不在定义信息中，则将其加入定义信息中，并使用给定的前置条件集合和存储的值初始化条件值集合。
   - 如果存储的地址已经在定义信息中，则检查每个条件值对，如果存储的值相同且前置条件不在条件值对中，则添加前置条件到条件值对中，并标记发生了改变。如果存储的值不存在于条件值对中，则将存储的值和前置条件添加到条件值对中，并标记发生了改变。
   
4. 返回一个布尔值，表示是否发生了改变。

### getCond

疑问：这段代码中，参数vars并没有被应用到，应用到的点被注释掉了

这段代码实现了从基本块对（BasicBlockPair）中获取条件节点（CondNode）的功能。具体步骤如下：

1. 首先，初始化条件节点的条件（cond）为nullptr，表示条件节点不存在，同时初始化trueBranch为false，表示条件的true分支。

2. 如果给定的起始基本块为nullptr，直接返回空的条件节点。

3. 如果起始基本块的终止指令是一个条件分支指令（BranchInst），则获取该指令的条件，并判断true分支的目标基本块是否等于给定的终止基本块。如果是，则将trueBranch设置为true。

4. 如果成功获取到条件（cond），则记录日志，并返回该条件和trueBranch。

5. 最终返回获取到的条件节点。

该函数用于从基本块对中提取条件信息，如果存在条件分支指令，则提取其条件以及true分支的信息，并返回为一个条件节点（CondNode）。



### containCondNode

在条件集合中查找给定条件，查看是否存在

### containCondNodeBranch

这段代码实现了在条件集合（Cond）中查找是否存在与给定的条件节点（CondNode）具有相同条件但满足情况不同的情况。具体步骤如下：

1. 首先，检查给定的条件节点是否为空，如果为空，则返回-1，表示未找到相同的条件节点。

2. 初始化索引idx为0，表示从条件集合的第一个条件开始检查。

3. 遍历条件集合中的每个条件节点，检查是否存在与给定的条件节点具有相同条件（first）但满足情况不同（second）的情况。如果找到符合条件的情况，则返回该条件节点的索引idx。

4. 如果遍历完所有条件节点仍未找到符合条件的情况，则返回-1，表示未找到相同的条件节点。

这段代码的作用是在条件集合中查找是否存在与给定的条件节点具有相同条件但满足情况不同的情况，并返回其在条件集合中的索引。

### sanitizeCond

合并操作，查看这个条件节点是否已经包含在了集合之中，如果包含

这段代码实现了从条件集合（Cond）中移除与给定的条件节点（CondNode）具有相同变量（vars）的条件节点。具体步骤如下：

1. 首先，使用 findCondNode 函数在条件集合中查找与给定的条件节点具有相同变量的条件节点的索引。

2. 如果找到符合条件的条件节点索引（idx 不为 -1），则表示存在与给定的条件节点具有相同变量的条件节点。于是，将该条件节点从条件集合中移除，并返回 true。

3. 如果未找到符合条件的条件节点，则返回 false，表示未进行任何移除操作。

该函数的作用是在条件集合中查找与给定的条件节点具有相同变量的条件节点，并将其从条件集合中移除。



