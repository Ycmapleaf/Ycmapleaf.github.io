# VLIW Packetizer


## DFA
[确定有限状态自动机 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/%E7%A1%AE%E5%AE%9A%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E8%87%AA%E5%8A%A8%E6%9C%BA)

DFA（deterministic finite automaton，确定有限状态自动机或确定有限自动机），一个能实现状态转移的[自动机](https://zh.wikipedia.org/wiki/%E8%87%AA%E5%8A%A8%E6%9C%BA "自动机")。对于一个给定的属于该自动机的状态和一个属于该自动机字母表的字符，它都能根据事先给定的转移函数转移到下一个状态（这个状态可以是先前那个状态）。

## DFAPacketizer
llvm/include/llvm/CodeGen/DFAPacketizer.h
llvm/lib/CodeGen/DFAPacketizer.cpp

[DFAPacketizer](../dfa_packetizer)

一个确定性有限自动机（DFA）由三个主要元素组成：状态（states）、输入（inputs）和转换（transitions）。对于打包机制来说，输入是目标指令类集合。状态模拟了在给定指令包中所有可能的功能单元消耗组合。转换模拟了将指令添加到指令包中的过程。在这个类构建的确定性有限自动机中，如果一个指令可以被添加到指令包中，那么就存在一个有效的从相应状态出发的转换。无效的转换表明该指令不能被添加到当前的指令包中。

## VLIWPacketizer
![DFA-VLIW.png](https://s2.loli.net/2024/09/03/csNGzXp9bOq3viK.png)

位于Post-RA之后，MC生成之前

调度边界：主要包括标签和终止符。

### 相关文件
llvm/lib/CodeGen/DFAPacketizer.cpp
llvm/include/llvm/CodeGen/DFAPacketizer.h

llvm/include/llvm/CodeGen/ScheduleDAGInstrs.h
llvm/lib/CodeGen/ScheduleDAGInstrs.cpp

### 相关名词
AAResults：Alias Analysis（别名分析）
SU：Scheduling unit，scheduling DAG上的节点

### 预处理
主要思想：去除基本块冗余部分，确定打包范围
1. 在打包前添加mutation。
2. 删除Kill伪指令。
3. 从基本块开始位置，找到第一个非调度边界RB。
4. 从3处位置开始，找到第一条调度边界RE。
5. 如果RE不为End，则自增
6. 如果RB不为End，根据打包策略对RB到RE内所有指令进行打包

### 打包
#### 生成inc
tablegen(LLVM MT3000GenDFAPacketizer.inc -gen-dfa-packetizer)

#### automaton
一个确定性有限状态自动机。该自动机在 TableGen 中定义；这个对象驱动由 tblgen 产生的表格定义的自动机。

自动机接受一系列输入标记（"动作"）。这个类是根据这些动作的类型来模板化的。

#### VLIWPacketizerList
##### 构造
1. **创建ResourceTracker**
   ResourceTracker = TII->CreateTargetScheduleState(MF.getSubtarget());
   目标后端需要覆盖**CreateTargetScheduleState**接口：
   - 该接口位于llvm/include/llvm/CodeGen/TargetInstrInfo.h。
   - 在目标后端的xxxInstrInfo中覆盖。
   - 该接口调用[生成inc](#生成inc)生成的createDFAPacketizer函数
2. **设定Transcribe**
   该属性确定是否打包器应该追踪指令和功能单元，false为只追踪指令
   - ResourceTracker->setTrackResources(true);
      - A.enableTranscription(Track);
         - Transcribe = Enable;
3. **创建VLIW调度器**
   VLIWScheduler = new DefaultVLIWScheduler(MF, mli, AA);
   该调度器会创建相关依赖图。

##### ResourceTracker
- getUsedResources
- clearResources
- canReserveResources

##### VLIWScheduler

#### 打包MI
##### VLIWScheduler准备
1. 确定开始基本块
2. 初始化DAG和调度器状态
3. 构建调度图

##### MI映射
将MI映射到SU上

##### 初始化打包器状态
initPacketizerState
需要在xxxPacketizer中覆盖

##### 单独打包指令
isSoloInstruction

##### 不需要打包的指令
ignorePseudoInstruction

### 解包
由于DFAPacketizer不提供解包接口，所以需要为相关目标提供解包功能。

实现解包接口`unpacketizeSoloInstrs`。

### 可视化
VLIWScheduler最终继承了llvm/include/llvm/CodeGen/ScheduleDAG.h的ScheduleDAG类

llvm/lib/CodeGen/ScheduleDAGPrinter.cpp包含ScheduleDAG::viewGraph方法

llvm/lib/CodeGen/MachineScheduler.cpp包含ScheduleDAGMI::viewGraph方法

llvm/lib/CodeGen/SelectionDAG/SelectionDAGPrinter.cpp包含SelectionDAG::viewGraph方法

### HazardRecognizer
目标后端的实现会在llvm/lib/CodeGen/PostRAHazardRecognizer.cpp中使用

### Others
llvm/lib/CodeGen/SelectionDAG/ScheduleDAGVLIW.cpp是对pre-RA的调度
llvm/lib/CodeGen/MachineScheduler.cpp是对machine的调度








