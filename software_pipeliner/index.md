# Software Pipeliner


**SMS：**
Swing Modulo Scheduling (SMS， 摇摆模调度) 是一种软件流水线技术，它旨在通过重叠循环的不同迭代指令来提高指令级并行性（ILP），从而优化循环的执行性能。SMS 特别关注于降低寄存器压力，同时在合理的编译时间内生成高效的调度序列 1。它属于模调度算法的一种，模调度算法尝试重叠单基本块循环的迭代，并根据一组启发式规则得出的优先级来调度指令。

**目标文件**：llvm/lib/CodeGen/MachinePipeliner.cpp

**Pass执行**：在寄存器分配Pass前。

| 概念                               | 含义                                                                                                          |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| MII(minimal initiation interval) | 指完成循环所需的最小间隔，MII=max(RecMII,ResMII)。                                                                        |
| ResMII(Resource MII)             | 根据一次循环所需的功能单元(FU, function unit)去除以机器所有功能单元的结果。如果循环中包含多类FU的使用(且每类FU之间相互无法替换)，则ResMII是分别对每一类FU计算ResMII后的最大值。 |
| RecMII(Recurrence MII)           | 循环中环路完成一次迭代所需的最小间隔，如果循环存在多条数据链路则分别计算后取其最大值。                                                                 |

## SMS算法
### 依赖图的计算和分析
依赖图包含四个部分：

| 属性        | 含义                                       |
| --------- | ---------------------------------------- |
| V         | 依赖图顶点的集合，其中每个顶点v表示循环中的一个操作。              |
| E         | 依赖图边的集合，表示依赖关系，SMS中仅存在数据依赖，(u,v)表示v依赖于u。 |
| $δ_(u,v)$ | 距离函数。                                    |
| $λ_u$     | 延迟函数，表示相应操作（顶点）所花费的周期数。                  |
附加函数：

| 属性                              | 含义                                     |
| ------------------------------- | -------------------------------------- |
| ASAP (As Soon As Possible)      | 指令可以开始执行的最早时间点。                        |
| ALAP (As Late As Possible)      | 指令可以开始执行的最晚时间点。                        |
| MOB(Mobility)<br>MOV (Movement) | 节点在调度中的移动量，是从ASAP到当前调度时间的差值，ALAP-ASAP。 |
| D (Depth)                       | 表示按延迟加权的节点的深度。                         |
| H (Height)                      | 表示按延迟加权的节点的高度。                         |
### 节点排序
**两种情况：**
- 无循环：自下而上遍历。
- 有循环：根据每个循环的RecMII的值，从高到低进行处理，遍历方式视情况而定。

**排序过程：**
- 计算部分排序/偏序（partial order），将节点分组为有序的集合列表：这些集合按优先级（RecMII值确定）从最高到最低的顺序排列，但每个集合内部没有任何顺序。图的每个节点只属于一个集合。
- 排序：按部分排序/偏序的优先级遍历每个节点集。
	- 确定该节点集的遍历方式（top-down/bottom-up）。
	- 根据遍历方式对节点进行遍历。
	- 最终输出包含图中所有节点的有序列表O。

![SMS_Ordering_Algorithm.png](https://s2.loli.net/2024/07/31/Q1azkD7WE5po8lx.png)

### 节点调度
| 属性        | 含义                                             |
| --------- | ---------------------------------------------- |
| $v$       | 前驱或后继操作节点。                                     |
| $u$       | 当前操作节点。                                        |
| $t_v$     | v已被调度的周期。                                      |
| $λ_v$     | v的延迟。                                          |
| $δ_(v,u)$ | v到u的依赖距离。                                      |
| $PSP(u)$  | previously scheduled predecessors，u已经被调度的前驱节点。 |
| $PSS(u)$  | previously scheduled successors，u已经被调度的后继节点。   |

- 如果在部分调度（partial schedule）中操作u仅有前驱节点，则尽可能早调度，计算开始时间`EarlyStart`，从`EarlyStart`开始扫描直到`EarlyStart+II-1`寻找空闲延迟槽。
  $EarlyStart = max_(v∈PSP(u))(t_v+λ_v-δ_(v,u)×II)$
- 如果在部分调度（partial schedule）中操作u仅有后继节点，则尽可能晚调度，计算开始时间`LateStart`，从`LateStart`开始扫描直到`LateStart+II-1`寻找空闲延迟槽。
  $LateStart = min_(v∈PSS(u))(t_v+λ_v-δ_(u,v)×II)$
- 如果在部分调度（partial schedule）中操作u既有前驱节点又有后继节点，计算`EarlyStart`和`LateStart`，从`EarlyStart`开始扫描直到`min(LateStart, EarlyStart + II - 1)`寻找空闲延迟槽。
- 如果在部分调度（partial schedule）中操作u既无前驱节点又无后继节点，则其`EarlyStart=ASAP`，从`EarlyStart`开始扫描直到`EarlyStart+II-1`寻找空闲延迟槽。

### 无循环示例
`MII = 4`

**依赖图示例：**

![SMS_Algorithm_Dependence_Graph_No_Circle.png](https://s2.loli.net/2024/07/31/cIx4oZUrH8jGAwV.png)

**各节点属性：**

![SMS_Algorithm_Node_Attr_No_Circle.png](https://s2.loli.net/2024/07/31/j9ce8wCIhSO5bPp.png)

**排序：**
- R = {n12}
- bottom-up：O = <n12, n11, n10, n8, n5, n6, n1, n2, n9>
- top-down：O = <n12, n11, n10, n8, n5, n6, n1, n2, n9, n3, n4, n7>

**调度：**

![SMS_Algorithm_Scheduling_No_Circle.png](https://s2.loli.net/2024/07/31/4EBStKHb91Whmlr.png)

### 有循环示例
四个通用功能单元、每个操作延迟为2。

`MII = 6`

**依赖图示例：**

![SMS_Algorithm_Dependence_Graph_Circle.png](https://s2.loli.net/2024/07/31/Io4jcdP1w5ZAVQT.png)

**排序：**
- 节点分组为有序集
	- S1 = {A, C, D, F}：第一个循环，$RecMII = (3 nodes × 2 cycles)/(1 distance) = 6$。
	- S2 = {G, J, M, I}：第二个循环，$RecMII = (3 nodes × 2 cycles)/(2 distance) = 3$。
	- S3 = {B, E, H, K, L}：剩余节点。
- 排序
	- S1：bottom-up：O = <F, C, D, A>
	- S2：top-down：O = <F, C, D, A, G, I, J, M>
	- S3：bottom-up：O = <F, C, D, A, G, I, J, M, H, E, B>
	- S3：top-down：O = <F, C, D, A, G, I, J, M, H, E, B, L, K>

**调度：**

![SMS_Algorithm_Scheduling_Circle.png](https://s2.loli.net/2024/07/31/4f6D7YAdMikPTrQ.png)

## llvm软流水实现流程
使用`ScheduleDAGInstrs`类，通过DAG结构表示依赖关系，使用指向 Phi 节点的顺序边表示循环相关性。

使用`DFAPacketizer`类，消除DAG中不必要的边，计算最小启动间隔，并检查在流水线计划中可以插入指令的位置。

1. 计算出最小初始间隔(MII, minimal initiation interval)。
2. 建立dependence graph, 计算每条指令的相关信息(ASAP ALAP MOV Height Depth ...)。
3. 节点排序。

## llvm软流水具体实现
`bool MachinePipeliner::scheduleLoop(MachineLoop &L)`在确切的循环上执行SMS算法，该函数识别候选循环，计算最小启动间隔，并尝试调度循环。

## llvm软流水相关符号
### dependence graph
- `SU`: Scheduling unit，调度单元
- `# preds left`: 当前调度单元剩余的前驱（predecessors）数量。
- `# succs left`: 当前调度单元剩余的后继（successors）数量。
- `# rdefs left`: 表示当前调度单元剩余的寄存器定义（register definitions）数量。
- `Latency`: 表示执行当前操作的延迟。
- `Depth`: 表示当前节点在控制流图中的深度。
- `Height`: 表示当前节点在控制流图中的高度。
- `Successors`: 列出了当前节点的后继节点。
- `SU(x): Data Latency=0 Reg=%y`: 表示到编号为x的后继调度单元的数据延迟是0，即使用寄存器`%y`的值没有延迟。
- `SU(x): Anti Latency=1`: 表示到编号为x的后继调度单元的反依赖（anti-dependence）延迟是1。反依赖通常发生在循环中，当一个指令的执行结果需要在某个时间点之前被另一个指令使用时。这里的“1”表示在当前节点和使用该结果的指令之间有一个周期的延迟。

### Node
| 属性                         | 含义                                     |
| -------------------------- | -------------------------------------- |
| ASAP (As Soon As Possible) | 指令可以开始执行的最早时间点。                        |
| ALAP (As Late As Possible) | 指令可以开始执行的最晚时间点。                        |
| MOV (Movement)             | 节点在调度中的移动量，是从ASAP到当前调度时间的差值，ALAP-ASAP。 |
| D (Depth)                  | 表示按延迟加权的节点的深度。                         |
| H (Height)                 | 表示按延迟加权的节点的高度。                         |
| ZLD (Zero Latency Depth)   | 表示在没有考虑指令延迟的情况下，节点的深度。                 |
| ZLH (Zero Latency Height)  | 表示在没有考虑指令延迟的情况下，节点的高度。                 |

### NodeSet
| 属性        | 含义                 |
| --------- | ------------------ |
| Num nodes | 当前节点集包含的节点数。       |
| rec       | RecMII。            |
| mov       | MaxMOV，节点在调度中的移动量。 |
| depth     | MaxDepth。          |
| col       | Colocate，定位。       |

## 参考
  1. Lifetime-Sensitive Modulo Scheduling in a Production Environment





