# DFA Packetizer


**本文章为LLVM DFAPacketizer的源码分析。**

相关文件路径
- llvm/include/llvm/CodeGen/DFAPacketizer.h
- llvm/lib/CodeGen/DFAPacketizer.cpp

一个确定性有限自动机（DFA）由三个主要元素组成：状态（states）、输入（inputs）和转换（transitions）。对于打包机制来说，输入是目标指令类集合。状态模拟了在给定指令包中所有可能的功能单元消耗组合。转换模拟了将指令添加到指令包中的过程。在这个类构建的确定性有限自动机中，如果一个指令可以被添加到指令包中，那么就存在一个有效的从相应状态出发的转换。无效的转换表明该指令不能被添加到当前的指令包中。

DFAPacketizer定义三个类`DefaultVLIWScheduler`、`DFAPacketizer`、`VLIWPacketizerList`。

## DefaultVLIWScheduler
构建依赖图。
### 类定义
```cpp
// 这个类扩展了 ScheduleDAGInstrs，并覆盖了 schedule 方法以构建依赖图。
class DefaultVLIWScheduler : public ScheduleDAGInstrs {
private:
  AAResults *AA;
  // DAG 后处理步骤的有序列表。
  std::vector<std::unique_ptr<ScheduleDAGMutation>> Mutations;

public:
  DefaultVLIWScheduler(MachineFunction &MF, MachineLoopInfo &MLI,
                       AAResults *AA);

  // 调度。
  void schedule() override;

  // DefaultVLIWScheduler拥有突变对象的所有权。
  void addMutation(std::unique_ptr<ScheduleDAGMutation> Mutation) {
    Mutations.push_back(std::move(Mutation));
  }

protected:
  void postProcessDAG();
};
```

### 变量
#### AA
别名分析。

`AAResults *AA`

#### Mutations
存储DAG后处理阶段的有序列表。

`std::vector<std::unique_ptr<ScheduleDAGMutation>> Mutations`
- ScheduleDAGMutation在DAG构建后为针对调度的依赖图的目标特定变异（llvm/include/llvm/CodeGen/ScheduleDAGMutation.h）。

### 目标无关函数
#### DefaultVLIWScheduler
构造函数。

#### schedule
调度工作。
```cpp
void DefaultVLIWScheduler::schedule() {
  // 构建调度图
  // llvm/lib/CodeGen/ScheduleDAGInstrs.cpp
  buildSchedGraph(AA);
  postProcessDAG();
}
```

#### addMutation
添加异变对象。

#### postProcessDAG
DAG后处理。

## DFAPacketizer
该类进行资源管理以及自动机操作。
### 类定义
```cpp
class DFAPacketizer {
private:
  const InstrItineraryData *InstrItins;
  // 自动机
  Automaton<uint64_t> A;
  // 对于每条调度，都有一个应用于自动机的“行为”。这消除了行程类别之间动作的冗余。
  ArrayRef<unsigned> ItinActions;

public:
  DFAPacketizer(const InstrItineraryData *InstrItins, Automaton<uint64_t> a,
                ArrayRef<unsigned> ItinActions)
      : InstrItins(InstrItins), A(std::move(a)), ItinActions(ItinActions) {
    // 开始时禁用资源跟踪功能。
    A.enableTranscription(false);
  }

  // 重置当前状态，另所有资源可获取
  void clearResources() {
    A.reset();
  }

  // 设置这个打包器是否不仅应该跟踪指令是否可以打包，
  // 而且还要跟踪在打包之后每条指令最终使用哪些功能单元。
  void setTrackResources(bool Track) {
    A.enableTranscription(Track);
  }

  // 检查当前状态下MCInstrDesc占用的资源是否可用。
  bool canReserveResources(const MCInstrDesc *MID);

  // 预留MCInstrDesc占用的资源，并改变当前状态以反映这一变化。
  void reserveResources(const MCInstrDesc *MID);

  // 检查机器指令占用的资源在当前状态下是否可用。
  bool canReserveResources(MachineInstr &MI);

  // 预留机器指令占用的资源，并更新当前状态以反映这一变化。
  void reserveResources(MachineInstr &MI);

  // 返回添加到这个数据包中的第InstIdx条指令所使用的资源。
  // 资源以功能单元的位向量形式返回。 
  // 注意，一个指令束可能有多种有效的打包方式。这个函数返回一种任意有效的打包。
  // 需要先调用setTrackResources(true)。
  unsigned getUsedResources(unsigned InstIdx);

  // 获取子目标提供的供目标使用的调度数据。
  const InstrItineraryData *getInstrItins() const { return InstrItins; }
};
```

### 变量
#### InstrItins
调度数据。

`const InstrItineraryData *InstrItins`

#### A
自动机。

`Automaton<uint64_t> A`


#### ItinActions
调度行为。

`ArrayRef<unsigned> ItinActions`


### 目标无关函数
#### DFAPacketizer
构造函数。
开启自动机A的转录功能（Transcription）。
#### clearResources
重置自动机A，使所有资源可获取。

#### setTrackResources
根据Track设置自动机A是否转录，表示除考虑打包外还应考虑功能单元的应用。

#### canReserveResources
1.`canReserveResources(const MCInstrDesc *MID)`: 检查被一个MCInstrDesc占用的资源是否可获取。
```cpp
bool DFAPacketizer::canReserveResources(const MCInstrDesc *MID) {
  unsigned Action = ItinActions[MID->getSchedClass()];
  if (MID->getSchedClass() == 0 || Action == 0)
    return false;
  return A.canAdd(Action);
}
```

```cpp
// llvm/include/llvm/Support/Automaton.h
bool canAdd(const ActionT &A) {
	auto I = M->find({State, A});
	return I != M->end();
}
```

- 根据MID获取调度类别。
- 根据调度类别获取调度行为。
- 如果没有调度类别或调度行为，结束。
- 否则使用自动机A确定调度行为是否可以被转换。

getSchedClass获取该指令的调度类别，其结果为[InstrItineraryData](InstrItineraryData.md)表的索引。


2.`canReserveResources(MachineInstr &MI)`: 检查MachineInstr占用的资源在当前状态下是否可用。

该方法调用`canReserveResources(const MCInstrDesc *MID)`。

#### reserveResources
1.`reserveResources(const MCInstrDesc *MID)`: 预留MCInstrDesc占用的资源，并改变当前状态以反映这一变化。
```cpp
void DFAPacketizer::reserveResources(const MCInstrDesc *MID) {
	unsigned Action = ItinActions[MID->getSchedClass()];
	if (MID->getSchedClass() == 0 || Action == 0)
		return;
	A.add(Action);
}
```

```cpp
// llvm/include/llvm/Support/Automaton.h
bool add(const ActionT &A) {
	auto I = M->find({State, A});
	if (I == M->end())
	  return false;
	if (Transcriber && Transcribe)
	  Transcriber->transition(I->second.second);
	State = I->second.first;
	return true;
}
```

- 根据MID获取调度类别。
- 根据调度类别获取调度行为。
- 如果没有调度类别或调度行为，结束。
- 否则将该行为添加到自动机A中。

2.`reserveResources(MachineInstr &MI)`: 预留MachineInstr占用的资源，并更新当前状态以反映这一变化。

该方法调用`reserveResources(const MCInstrDesc *MID)`。

#### getUsedResources
返回添加到这个数据包中的第InstIdx条指令所使用的资源，资源以功能单元的位向量形式返回。注意，一个指令束可能有多种有效的打包方式。这个函数返回一种任意有效的打包。需要先调用setTrackResources(true)。

```cpp
unsigned DFAPacketizer::getUsedResources(unsigned InstIdx) {
  ArrayRef<NfaPath> NfaPaths = A.getNfaPaths();
  assert(!NfaPaths.empty() && "Invalid bundle!");
  const NfaPath &RS = NfaPaths.front();

  // RS stores the cumulative resources used up to and including the I'th
  // instruction. The 0th instruction is the base case.
  if (InstIdx == 0)
    return RS[0];
  // Return the difference between the cumulative resources used by InstIdx and
  // its predecessor.
  return RS[InstIdx] ^ RS[InstIdx - 1];
}
```

#### getInstrItins
获取子目标提供的供目标使用的调度数据

## VLIWPacketizerList
VLIWPacketizerList 实现了一个使用 DFA（确定性有限自动机）的简单 VLIW 指令打包器。该打包器在机器基本块上工作。对于 BB（基本块）中的每条指令 I，打包器会查询 DFA 来看是否有足够的机器资源来执行 I。如果是这样，打包器会检查 I 是否依赖当前数据包中的任何指令。如果没有发现依赖关系，I 就会被添加到当前数据包中，并且相应的机器资源会被标记为已占用。如果发现了依赖关系，就会进行目标 API 调用以剪枝依赖。
### 类定义
```cpp
class VLIWPacketizerList {
protected:
  MachineFunction &MF;
  const TargetInstrInfo *TII;
  AAResults *AA;

  // VLIW调度器
  DefaultVLIWScheduler *VLIWScheduler;
  // 当前数据包包含的指令。
  std::vector<MachineInstr*> CurrentPacketMIs;
  // DFA资源追踪器。
  DFAPacketizer *ResourceTracker;
  // 指令到功能单元的映射。
  std::map<MachineInstr*, SUnit*> MIToSUnit;

public:
  // 构造函数，AAResults参数可以为空指针。
  VLIWPacketizerList(MachineFunction &MF, MachineLoopInfo &MLI,
                     AAResults *AA);

  virtual ~VLIWPacketizerList();

  // 指令打包接口。
  void PacketizeMIs(MachineBasicBlock *MBB,
                    MachineBasicBlock::iterator BeginItr,
                    MachineBasicBlock::iterator EndItr);

  // 返回ResourceTracker。
  DFAPacketizer *getResourceTracker() {return ResourceTracker;}

  // 添加指令到当前包。
  virtual MachineBasicBlock::iterator addToPacket(MachineInstr &MI) {
    CurrentPacketMIs.push_back(&MI);
    ResourceTracker->reserveResources(MI);
    return MI;
  }

  // 结束当前打包并且重置打包器的状态。
  // 覆盖当前函数允许确切目标打包器执行自定义最终处理。
  virtual void endPacket(MachineBasicBlock *MBB,
                         MachineBasicBlock::iterator MI);

  // 在将指令打包之前执行初始化。该函数应由依赖于目标的打包器覆盖。
  virtual void initPacketizerState() {}

  // 检查是否给定的指令应当被打包器忽视。
  virtual bool ignorePseudoInstruction(const MachineInstr &I,
                                       const MachineBasicBlock *MBB) {
    return false;
  }

  // 若当前指令不能和任意一个指令打包，则返回true，这意味着当前指令独自为一个包。
  virtual bool isSoloInstruction(const MachineInstr &MI) { return true; }

  // 检查打包器是否应该尝试将给定的指令添加到当前数据包中。
  // 可能不希望将指令包含在当前数据包中的原因之一是，它可能导致停滞。 
  // 如果这个函数返回 "false"，则当前数据包将结束，并且该指令将被添加到下一个数据包中。
  virtual bool shouldAddToPacket(const MachineInstr &MI) { return true; }

  // 检查将 SUI 和 SUJ 打包在一起是否合法。
  virtual bool isLegalToPacketizeTogether(SUnit *SUI, SUnit *SUJ) {
    return false;
  }

  // 检查在 SUI 和 SUJ 之间剪枝是否合法。
  virtual bool isLegalToPruneDependencies(SUnit *SUI, SUnit *SUJ) {
    return false;
  }

  // 在打包开始前，增加一次 DAG 突变。
  void addMutation(std::unique_ptr<ScheduleDAGMutation> Mutation);

  bool alias(const MachineInstr &MI1, const MachineInstr &MI2,
             bool UseTBAA = true) const;

private:
  bool alias(const MachineMemOperand &Op1, const MachineMemOperand &Op2,
             bool UseTBAA = true) const;
};
```

### 变量
#### MF

`MachineFunction &MF`

#### TII
目标指令信息。

`const TargetInstrInfo *TII`

#### AA
别名分析。

`AAResults *AA`

#### VLIWScheduler
VLIW调度器。

`DefaultVLIWScheduler *VLIWScheduler`

#### CurrentPacketMIs
当前数据包包含的指令

`std::vector<MachineInstr*> CurrentPacketMIs`

#### ResourceTracker
DFA资源追踪器。

`DFAPacketizer *ResourceTracker`

#### MIToSUnit
指令到功能单元的映射关系。

`std::map<MachineInstr*, SUnit*> MIToSUnit`


### 目标无关函数
#### VLIWPacketizerList
构造函数，AAResults参数可以为空指针。

#### PacketizeMIs
指令打包接口。
```cpp
void VLIWPacketizerList::PacketizeMIs(MachineBasicBlock *MBB,
                                      MachineBasicBlock::iterator BeginItr,
                                      MachineBasicBlock::iterator EndItr) {
  assert(VLIWScheduler && "VLIW Scheduler is not initialized!");
  // 准备调度执行。
  VLIWScheduler->startBlock(MBB);
  // 为新调度区域初始化 DAG 和通用调度器状态。
  // 这实际上并不创建 DAG，只是清除它。
  // 调度驱动程序可在每个调度区域多次调用 BuildSchedGraph。
  VLIWScheduler->enterRegion(MBB, BeginItr, EndItr,
                             std::distance(BeginItr, EndItr));
  VLIWScheduler->schedule();

  LLVM_DEBUG({
    dbgs() << "Scheduling DAG of the packetize region\n";
    VLIWScheduler->dump();
  });

  // 生成机器指令到调度单元的映射。
  MIToSUnit.clear();
  for (SUnit &SU : VLIWScheduler->SUnits)
    MIToSUnit[SU.getInstr()] = &SU;

  bool LimitPresent = InstrLimit.getPosition();

  // 打包。
  for (; BeginItr != EndItr; ++BeginItr) {
    if (LimitPresent) {
      if (InstrCount >= InstrLimit) {
        EndItr = BeginItr;
        break;
      }
      InstrCount++;
    }
    MachineInstr &MI = *BeginItr;
    initPacketizerState();

    // 如果需要结束当前打包。
    if (isSoloInstruction(MI)) {
      endPacket(MBB, MI);
      continue;
    }

    // 忽视伪指令。
    if (ignorePseudoInstruction(MI, MBB))
      continue;

    SUnit *SUI = MIToSUnit[&MI];
    assert(SUI && "Missing SUnit Info!");

    // 询问DFA该机器指令所需资源是否可获取。
    LLVM_DEBUG(dbgs() << "Checking resources for adding MI to packet " << MI);

    bool ResourceAvail = ResourceTracker->canReserveResources(MI);
    LLVM_DEBUG({
      if (ResourceAvail)
        dbgs() << "  Resources are available for adding MI to packet\n";
      else
        dbgs() << "  Resources NOT available\n";
    });
    if (ResourceAvail && shouldAddToPacket(MI)) {
      // 检查当前指令和包中指令依赖关系。
      for (auto *MJ : CurrentPacketMIs) {
        SUnit *SUJ = MIToSUnit[MJ];
        assert(SUJ && "Missing SUnit Info!");

        LLVM_DEBUG(dbgs() << "  Checking against MJ " << *MJ);
        // 打包SUI和SUJ是否合法。
        if (!isLegalToPacketizeTogether(SUI, SUJ)) {
          LLVM_DEBUG(dbgs() << "  Not legal to add MI, try to prune\n");
          // 如果依赖可以被剪枝则打包。
          if (!isLegalToPruneDependencies(SUI, SUJ)) {
            // 如果依赖不能剪枝则结束打包。
            LLVM_DEBUG(dbgs()
                       << "  Could not prune dependencies for adding MI\n");
            endPacket(MBB, MI);
            break;
          }
          LLVM_DEBUG(dbgs() << "  Pruned dependence for adding MI\n");
        }
      }
    } else {
      LLVM_DEBUG(if (ResourceAvail) dbgs()
                 << "Resources are available, but instruction should not be "
                    "added to packet\n  "
                 << MI);
      // 如果资源不可获取或指令不能加入当前包则结束打包。
      endPacket(MBB, MI);
    }

    // 添加MI到当前包。
    LLVM_DEBUG(dbgs() << "* Adding MI to packet " << MI << '\n');
    BeginItr = addToPacket(MI);
  } // 对于打包范围的所有指令。

  // 结束任何遗留的数据包。
  endPacket(MBB, EndItr);
  VLIWScheduler->exitRegion();
  VLIWScheduler->finishBlock();
}
```

#### getResourceTracker
返回ResourceTracker。

#### addMutation
在打包开始前，增加一次 DAG 突变。

#### alias
`public`

#### alias
`private`

### 目标相关函数
#### ~VLIWPacketizerList()
析构函数

#### addToPacket
添加指令到当前包。**（存在默认实现）**

#Todo 

#### endPacket
结束当前打包并且重置打包器的状态，覆盖当前函数允许确切目标打包器执行自定义最终处理。**（存在默认实现）**

#Todo 

#### initPacketizerState
在将指令打包之前执行初始化。该函数应由依赖于目标的打包器覆盖。

#Todo 

#### ignorePseudoInstruction
检查是否给定的指令应当被打包器忽视。

#Todo 

#### isSoloInstruction
若当前指令不能和任意一个指令打包，则返回true，这意味着当前指令独自为一个包。

#Todo 

#### shouldAddToPacket
检查打包器是否应该尝试将给定的指令添加到当前数据包中。
可能不希望将指令包含在当前数据包中的原因之一是，它可能导致停滞。 
如果这个函数返回 "false"，则当前数据包将结束，并且该指令将被添加到下一个数据包中。

#Todo 

#### isLegalToPacketizeTogether
检查将 SUI 和 SUJ 打包在一起是否合法。

#Todo 

#### isLegalToPruneDependencies
检查剪枝 SUI 和 SUJ 之间的依赖是否合法。

#Todo 

