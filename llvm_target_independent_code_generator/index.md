# Llvm_target_independent_code_generator


[The LLVM Target-Independent Code Generator — LLVM 15.0.0 documentation](https://releases.llvm.org/15.0.0/docs/CodeGenerator.html#implementations-of-the-abstract-target-description-interfaces)
是一个提供一套可复用的组件的框架，将LLVM内部表示以汇编形式（静态编译器）、二进制（JIT编译器）翻译成特定平台的机器码
·抽象目标描述接口 in `include/llvm/Target/`
·用于表示为目标生成的代码的类。这些类旨在足够抽象，以表示任何目标机器的机器代码 in `include/llvm/CodeGen/`
·用于在目标文档级别（MC层）表示代码的类和算法。这些类表示汇编级构造，如标签、节和指令
·目标无关的算法，用于实现本机代码生成的各个阶段（寄存器分配、调度、堆栈帧表示等） in `lib/CodeGen/`
·特定目标的抽象目标描述接口的实现。这些机器描述利用 LLVM 提供的组件，并可以选择提供特定于目标的自定义pass，为特定目标构建完整的代码生成器 in `lib/Target/`
·目标无关的JIT组件 in `lib/ExecutionEngine/JIT`

## Target description classes
独立于任何客户端的目标机器的抽象描述

### TargetMachine
目标描述类的实现，需要继承
提供`get*Info`虚拟方法去访问目标描述类
getInstrInfo
getRegisterInfo
getFrameInfo
...

### DataLayout
数据布局，唯一且必需，不能派生
内存结构、数据类型对齐方式、指针大小、大端序或小端序

### TargetLowering
如何将LLVM Code降为SelectionDAG操作
定义：
·目标机器所支持操作
·用于位移数量的类型
·`setcc`操作的返回类型
·高级特征，比如是否将除法变为乘法序列

### TargetRegisterInfo
寄存器描述、寄存器和寄存器之间的交互
虚拟寄存器：无符号整型，大数字
物理寄存器：唯一小数字
寄存器`#0`表示标志值
每个寄存器有一个相关联的`TargetRegisterDesc`，表明寄存器名称和别名
该类公开一组特定处理器的寄存器类，每一个寄存器类包含了一组寄存器，他们有相同的属性。每一个由指令选择器创建的SSA虚拟寄存器都和一个寄存器类相关，当寄存器分配器运行时，会将这些虚拟寄存器以物理寄存器进行代替。==这些寄存器类的实现是由tablegen文件自动生成的==

### TargetInstrInfo
描述目标支持的机器指令
·操作码的助记符
·操作数的数量
·隐式寄存器的使用和定义列表
·指令是否有与目标无关的属性（访问内存、可互交换等）
·任何特定于目标的标志

### TargetFrameLowering
目标堆栈结构的布局
保存堆栈增长的方向、进入每个函数时的已知堆栈对齐以及局部区域的偏移量，局部区域的偏移量是从函数入口上的堆栈指针到可以存储函数数据（局部变量、溢出位置）的第一个位置的偏移量

### TargetSubTarget
目标芯片组信息
子目标通知代码生成支持哪些指令、指令延迟和指令执行路线;即，使用哪些处理单元、以什么顺序以及使用多长时间

### TargetJITInfo
提供一个抽象接口，该接口使用JIT（Just-In-Time及时）代码生成去执行目标确切的活动

## Machine code description classes
在高级别，LLVM代码被转换为由MachineFunction，MachineBasicBlock和MachineInstr实例组成的机器特定表示，这个表示完全与目标无关，它以一种抽象形式：一个操作码和一系列操作数表示指令
这种表示旨在支持机器码的SSA表示、寄存器分配、非SSA形式

### MachineInstr
目标机器的指令表示为MachineInstr类的实例，此类只跟踪一个操作码和一组操作数
操作码是一个简单的无符号整型，仅仅对特定后端有意义
一个目标后端的指令定义在`*InstrInfo.td`中，操作码的枚举值是根据此文件自动生成的，MachineInstr类没有关于怎样解释指令的信息，需要参考[TargetInstrInfo](#TargetInstrInfo)类
机器指令的操作数可以是几种不同的类型：寄存器引用、常量整数、基本块引用等。机器操作数应标记为def或值的使用（尽管只允许寄存器为defs）
按照惯例，LLVM代码生成器对指令操作数进行排序，以便所有寄存器定义都出现在寄存器使用之前，即使在通常以其他顺序打印的体系结构上也是如此。例如，SPARC 添加指令：“add %i1， %i2， %i3”，“%i1”，“%i2” 寄存器值相加并将结果存储到 “%i3” 寄存器中。在 LLVM 代码生成器中，操作数应存储为 “%i3， %i1， %i2”：目标在第一位。这样方便调试输出以及创建仅仅def第一个操作数的指令
#### MachineInstrBuilder.h
BuildMI：便于创建任意机器指令，MachineInstruction
```cpp
// Create a 'DestReg = mov 42' (rendered in X86 assembly as 'mov DestReg, 42')
// instruction and insert it at the end of the given MachineBasicBlock.
const TargetInstrInfo &TII = ...
MachineBasicBlock &MBB = ...
DebugLoc DL;
MachineInstr *MI = BuildMI(MBB, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create the same instr, but insert it before a specified iterator point.
MachineBasicBlock::iterator MBBI = ...
BuildMI(MBB, MBBI, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create a 'cmp Reg, 0' instruction, no destination reg.
MI = BuildMI(MBB, DL, TII.get(X86::CMP32ri8)).addReg(Reg).addImm(42); 

// Create an 'sahf' instruction which takes no operands and stores nothing.
MI = BuildMI(MBB, DL, TII.get(X86::SAHF));

// Create a self looping branch instruction.
BuildMI(MBB, DL, TII.get(X86::JNE)).addMBB(&MBB);
```

#### Fixed（preassigned） register
固定寄存器
EAX、EBX ...

#### Call-clobbered registers
调用、中断寄存器
相较于每次添加<def,dead>操作数，不如只使用一个MO_RegisterMask操作数代替，寄存器掩码操作数包含保留寄存器的位掩码，其他所有内容都被视为被指令破坏

#### Machine code in SSA form
MachineInstr最初以SSA形式选择，并以SSA形式维护，直到发生寄存器分配。在大多数情况下，这非常简单，因为 LLVM 已经是 SSA 形式；LLVM PHI 节点成为机器码 PHI 节点，虚拟寄存器只允许有一个定义
寄存器分配后，机器代码不再是 SSA 形式，因为代码中没有剩余的虚拟寄存器。

### MachineBasicBlock
该类包含一个机器指令的列表（==MachineInstr实例==），它大致对应于指令选择器的LLVM代码输入，但可以有一对多映射（即一个LLVM基本块可以映射到多个机器基本块）
包含`getBasicBlock`方法，返回来自的LLVM基本块

### MachineFunction
`include/llvm/CodeGen/MachineFunction.h`
该类包含一个机器基础块的列表（==MachineBasicBlock的实例==），它和指令选择器的LLVM函数输入一一对应
除此之外，该类包含一个 `MachineConstantPool`, 一个 `MachineFrameInfo`, 一个 `MachineFunctionInfo`, 和一个 `MachineRegisterInfo`

### MachineInstr Bundles
#obscure
```text
-------------- 
|   Bundle   | ---------
--------------          \
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |
--------------
|   Bundle   | --------
--------------         \
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |                  ...
       |
--------------
|   Bundle   | --------
--------------         \
       |
      ...
```

MachineInstr passes应该作为单个单元在MI bundle上操作

## MC Layer
MC 层用于在原始机器代码级别表示和处理代码，没有“高级”信息，如“常量池”、“跳转表”、“全局变量”或类似的东西。

这一层中的代码用于许多重要目的：代码生成器的尾端使用它来编写.s或.o文档，并且LLVM-mc工具也使用它来实现独立的机器代码汇编器和反汇编器。

### MCStreamer
#obscure 
将LLVM IR转换为目标机器码（机器指令）

它是一个抽象的API，以不同的方式实现（例如，输出.s文档，输出ELF .o文档等），但其API直接对应于在.s文档中看到的内容。

MCStreamer每个指令都有一个方法，如EmitLabel，EmitSymbolAttribute，switchSection，emitValue（用于.byte，.word）等，直接对应于汇编级指令。它也有一个EmitInstruction方法，输出一个MCInst到流中

两种实现：
·输出.s文件（MCAsmStreamer）
·输出.o文件（MCObjectStreamer）
MCAsmStreamer是一个简单的实现，它为每个方法打印出一个指令（例如EmitValue -> .byte），但MCObjectStreamer实现了完整的汇编进程。

对于目标确切的指令，MCStreamer有一个MCTargetStreamer实例，每一个需要它的目标定义了一个继承它的类，每个指令包含一个方法两个继承它的类，分别是target object streamer 和  target asm streamer。target asm streamer只打印它（emitFnStart -> .fnstart），target object streamer为它实现汇编逻辑。

为了使llvm调用这些类，目标必须调用TargetRegistry::RegisterAsmStreamer 和 TargetRegistry::RegisterMCObjectStreamer传递回调，这些回调分配相应的目标流并将其传递给 createAsmStreamer 或相应的对象流构造函数。

### MCContext
MCContext 类是 MC 层各种唯一数据结构的所有者。
因此，这是可以与之交互以创建symbols和sections的类。此类不能被子类化。

### MCSymbol
在汇编文件中表示一个标签
·汇编临时标签
被汇编器使用但会在目标文件生成时丢弃
·普通标签

两种的区别通常通过在标签中添加前缀来表示，例如“L”标签是MachO中的汇编临时标签。

MCSymbols由MCContext创建并且是唯一的，可以通过比较指针的等价性来判断是否为相同的symbol，指针不等不能保证标签最终会位于不同地址

### MCSection
代表目标文件确切的section

被目标文件进行子类化来实现（`MCSectionMachO`, `MCSectionCOFF`, `MCSectionELF`），由由MCContext创建并且是唯一的

MCStreamer具有当前部分的概念，可以使用SwitchToSection方法（对应于.s文档中的“.section”指令）进行更改。

### MCInst
是机器指令的目标无关表示（MC层）
比MachineInstr更简单的类，包含明确目标的操作码和MC操作数向量
是指令编码器、指令输出使用的类型，以及汇编解析器和反汇编器生成的类型。

MCOperand有三种情况：
1）一个简单的立即数
2）目标寄存器ID
3）作为MCExpr的符号表达式（例如“Lfoo-Lbar+42”）

### Object File Format

|Format|Supported Targets|
|---|---|
|`COFF`|AArch64, ARM, X86|
|`DXContainer`|DirectX|
|`ELF`|AArch64, AMDGPU, ARM, AVR, BPF, CSKY, Hexagon, Lanai, LoongArch, M86k, MSP430, MIPS, PowerPC, RISCV, SPARC, SystemZ, VE, X86|
|`GCOFF`|SystemZ|
|`MachO`|AArch64, ARM, X86|
|`SPIR-V`|SPIRV|
|`WASM`|WebAssembly|
|`XCOFF`|PowerPC|

## Target-independent code generation algorithms
描述代码生成的高级设计，解释如何工作以及设计背后的逻辑合理性

### Instruction Selection
将呈现给代码生成器的LLVM code转化为目标确切机器指令
DAG指令选择器是从目标描述（.td）文件中生成的

#### SelectionDAG
操作节点类型描述 in `include/llvm/CodeGen/ISDOpcodes.h`

#### SelectionDAG Instruction Selection Process
每次合法化之后需要优化，主要优化插入标志和零扩展指令
#### Build initial DAG 
将LLVM code输入转化为非法的SelectionDAG
此pass的目的是向 SelectionDAG 公开尽可能多的低级别、特定于目标的详细信息。需要特定于目标的钩子来降低调用、返回、varargs 等
对于这些特征，使用[TargetLowering](#TargetLowering)接口

#### Optimize SelectionDAG
简化：使用简单优化方式简化DAG
识别：识别支持这些元操作的目标上的元指令（例如旋转和除/余数对）
使生成的代码更高效，select instructions from DAG 阶段的选择指令更简单

#### Legalize SelectionDAG Types 
转换SelectionDAG节点以消除任何目标不支持的类型

标量：promoting（小类型提升为大类型）、expanding（大类型分解为小类型）
向量：widening（将向量多次拆分）、scalarizing（标量化）

目标实现通过在其 TargetLower 构造函数中调用 addRegisterClass 方法来告诉合法化进程支持哪些类型（以及用于它们的寄存器类）

#### Optimize SelectionDAG 
清理类型合法化带来的冗余

#### Legalize SelectionDAG Ops
转换SelectionDAG节点以消除任何目标不支持的操作
expansion
promotion
custom

shufflevector向量重排

#### Optimize SelectionDAG 
消除操作合法化带来的低效率

#### Select instructions from DAG 
将目标无关DAG输入转换为特定目标指令的DAG
```
%t1 = fadd float %W, %X
%t2 = fmul float %t1, %Y
%t3 = fadd float %t2, %Z

<-->(fadd:f32 (fmul:f32 (fadd:f32 W, X), Y), Z)
如果目标支持FMADDS，即乘加指令，转换为
-->(FMADDS (FADDS W, X), Y, Z)
```

```text
def FMADDS : AForm_1<59, 29,
                    (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRC, F4RC:$FRB),
                    "fmadds $FRT, $FRA, $FRC, $FRB",
                    [(set F4RC:$FRT, (fadd (fmul F4RC:$FRA, F4RC:$FRC),                                           F4RC:$FRB))]>;
def FADDS : AForm_2<59, 21,
                    (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRB),
                    "fadds $FRT, $FRA, $FRB",
                    [(set F4RC:$FRT, (fadd F4RC:$FRA, F4RC:$FRB))]>;
```
F4RC是输入和结果的寄存器类
DAG操作 in `include/llvm/Target/TargetSelectionDAG.td`
TableGen DAG指令选择生成器读取`.td`文件中的`pattern`并自动构建模式匹配代码：
·编译时，分析指令模式是否有意义
·处理模式匹配的操作数上的任意约束
·自动类型推断
·目标可以定义自己的（并依赖于内置的）模式片段
模式片段是可重用的模式的块，在编译时内联到模式中
·使用`Pat`类定义pattern对应一个或多个指令
```text
def : Pat<(i32 imm:$imm),
          (ORI (LIS (HI16 imm:$imm)), (LO16 imm:$imm))>;
将32位立即数放入寄存器中
匹配过程：
接收一个32位立即数
ORI:表示`or`操作
LIS:将16位立即数左移16位
HI16和LO16拿取32位立即数的高16和低16位
```
#### SelectionDAG Scheduling and Formation 
为目标指令DAG中的指令分配一个线性顺序并发送其到正在编译的[MachineFunction](#MachineFunction)中
从选择阶段得到目标指令的DAG并分配顺序，当排好序后将DAG转化为一个MachineInstr列表

完成所有这些步骤后，将销毁 SelectionDAG 并运行其余代码生成过程


### SSA-based Machine Code Optimizations


### Live Intervals
活动时期
在寄存器分配pass中决定是否需要同一个物理寄存器的两个或更多的虚拟寄存器在程序中的同一点处于活动状态，如果存在，将一个寄存器溢出

#### Live Variable Analysis
#obscure

#### Live Intervals Analysis
#obscure

### Register Allocation
虚拟寄存器无限，物理寄存器有限。
如果物理寄存器不能适应所有虚拟寄存器，则将部分映射到内存中，被称为溢出虚拟对象

#### Registers represent in LLVM
物理寄存器：1-1023 in `GenRegisterNames.inc`
物理寄存器别名：RegisterInfo.td  MCRegAliasIterator
LLVM 中的物理寄存器按寄存器类分组。同一寄存器类中的元素在功能上是等效的，并且可以互换使用
不同物理寄存器可能使用相同编号
静态定义在TargetRegisterInfo.td中

不同虚拟寄存器不会使用相同编号
使用`MachineRegisterInfo::createVirtualRegister()`创建新的虚拟寄存器
`MachineOperand::isRegister()`：是否是寄存器
`MachineOperand::getReg()`：得到寄存器编号
`MachineOperand::isUse()`：是否被指令使用
`MachineOperand::isDef()`：是否定义

我们将在寄存器分配之前LLVM位码中存在的物理寄存器称为预着色寄存器（Pre-colored）
·传递函数调用参数
·存储特殊指令结果
分为：
·隐式定义：静态定义在每个指令
·显式定义：依赖被编译的程序
预着色寄存器对任何寄存器分配算法施加约束。寄存器分配器必须确保它们都不会在虚拟寄存器处于活动状态时被虚拟寄存器的值覆盖。

#### Mapping virtual registers to physical registers
·direct mapping
使用`TargetRegisterInfo`和`MachineOperand`类
利于寄存器分配的开发人员，但更容易出错，并需要大量工作实现。
程序员必须指定在正在编译的目标函数中应插入加载和存储指令的位置，以便在内存中获取和存储值。
要将物理寄存器分配给给定操作数中存在的虚拟寄存器，使用 MachineOperand：：setReg（p_reg）。要插入存储指令，使用 TargetInstrInfo：：storeRegToStackSlot（...），要插入加载指令，使用 TargetInstrInfo：：loadRegFromStackSlot。

·indirect mapping
使用`VirtRegMap`类，插入加载和存储向内存发送和从内存获取值。
间接映射使应用程序开发人员免受插入加载和存储指令的复杂性的影响。
为了将虚拟寄存器映射到物理寄存器，使用 VirtRegMap::assignVirt2Phys(vreg, preg)。为了将某个虚拟寄存器映射到内存，使用 VirtRegMap::assignVirt2StackSlot(vreg)

#### Handling two address instructions
大多数LLVM机器码指令是三地址指令，即至多定义一个寄存器，至少使用两个寄存器
少部分结构使用二地址指令，则被定义的寄存器即被定义也被使用
将代表二地址指令的三地址指令转化为二地址指令：`TwoAddressInstructionPass`，其在寄存器分配前执行并替代三地址指令。但是执行后的指令不符合SSA形式
```text
%a = ADD %b %c
	↓
%a = MOVE %b
%a = ADD %a %c
```

#### The SSA deconstruction phase
是寄存器分配阶段的重要转换
SSA形式简化了对程序控制流图执行的许多分析，但是传统指令集不能实现PHI（是SSA的）指令，为了生成可执行代码，编译器必须将PHI指令替换为其他可保留语义的指令
·最传统的 PHI 解构算法用复制指令取代 PHI 指令，in `lib/CodeGen/PHIElimination.cpp`。
需要在寄存器分配器中标记`PHIEliminationID`标识符

#### Instruction folding
一种在寄存器分配阶段移除不必要的复制指令的优化
```text
%EBX = LOAD %mem_address
%EAX = COPY %EBX

%EAX = LOAD %mem_address
```
使用`TargetRegisterInfo::foldMemoryOperand(...)`方法折叠指令，一个指令折叠前后有很大差异
在`lib/CodeGen/LiveIntervalAnalysis.cpp`的`LiveIntervals::addIntervalsForSpills`中有相关例子

#### Built in register allocators
LLVM提供了三种寄存器分配器：
·Fast
调试建立默认分配器，在基础块级别，保留寄存器值并尽可能重复利用寄存器
·Basic
实时范围按启发式驱动的顺序一次分配给一个寄存器
·Greedy
默认分配器，此分配器努力将溢出代码的成本降至最低。
·PBQP
Partitioned Boolean Quadratic Programming
分段布尔二次规划
此分配器的工作原理是构造一个表示所考虑的寄存器分配问题的 PBQP 问题，使用 PBQP 求解器解决此问题，然后将解决方案映射回寄存器分配。

### Prolog/Epilog Code Insertion


### Compact Unwind
抛出异常需要展开一个函数，怎样展开给定函数的信息通常使用DWARF表示，但是每个函数每一个FDE需要20~30字节
DWARF：Debugging With Attributed Record Formats
FDE：Frame Description Entry 帧描述条目

compact unwind每一个函数只需要4字节表示-32bit
它指定要恢复哪些寄存器以及从何处恢复，以及展开函数。

当链接时，会创建一个`__TEXT,__unwind_info`部分，这个section轻量并且能很快运行去获取函数展开信息。
如果使用compact unwind，会将其编码到`__TEXT,__unwind_info`
如果使用DWARF unwind，在链接时`__TEXT,__unwind_info`会包含`__TEXT,__eh_frame`，其中包含了FDE的偏移量


### Late Machine Code Optimizations


### Code Emission
代码生成的代码发射步骤负责从代码生成器抽象（如MachineFunction，MachineInstr等）降低到MC层使用的抽象（MCInst，MCStreamer等），这是由多种类结合完成的：与目标无关的 AsmPrinter 类、AsmPrinter 的目标特定子类（如 SparcAsmPrinter）和 TargetLoweringObjectFile 类。

MC layers在目标文件的抽象级别工作，它没有函数、全局变量等的概念。它考虑的是标签、命令、指令。此时使用的关键类是MCStreamer，这是一个抽象的API，以不同的方式实现（例如输出.s文档，输出ELF .o文档等），实际上是一个“汇编进程API”

为Target实现code generator：
·为目标定义AsmPrinter的子类
·为目标实现指令打印器，指令打印器将一个MCInst作为文本发送到raw_osream，多数是由.td文件自动生成的
·实现MachineInstr到MCInst的代码 in `<target>MCInstLower.cpp`，负责将跳转表条目、常量池索引、全局变量地址等转换为MCLabels，也负责将代码生成器使用的伪操作扩展为相对应的实际机器指令。由此产生的MCInst被送入指令打印器或编码器。

可以实现一个MCCodeEmitter的子类将MCInst降低为机器代码字节并重定位

### VLIW Packetizer
Very Long Instruction Word
在超长指令字 （VLIW） 体系结构中，编译器负责将指令映射到体系结构上可用的功能单元。为此，编译器创建称为数据包或捆绑包的指令组。LLVM 中的 VLIW 数据包器是一种独立于目标的机制，用于启用机器指令的数据包化。

#### Mapping from instructions to functional units
VLIW目标可以被映射为多个函数单元
在数据打包过程中，需要确定指令是否可以放入包中，通过检查所有可能映射来确定，相对复杂。VLIW Packetizer在编译器build时通过解析目标指令类并生成表格来降低复杂度，可以通过提供的机器无关API去询问这些表格来决定是否指令可以容纳到包中

#### How the packetization tables are generated and used
packetizer从目标Itinerary中读取指令类并创建DFA

DFA：deterministic finite automaton 确定有限自动机
·inputs：表示要添加到包中的指令
·states：表示包中的指令可能消耗的函数单元
·transitions：添加指令到已存在的包，如果指令到函数单元映射合法，则会出现相对应的transition，没有transition表示不存在合法映射并且指令不能被添加到包中

要为 VLIW 目标生成表，将 TargetGenDFAPacketizer.inc 作为目标添加到目标目录中的Makefile中。导出的 API 提供三个函数：`DFAPacketizer：：clearResources（）`、`DFAPacketizer：：reserveResources（MachineInstr *MI）` 和 `DFAPacketizer：：canReserveResources（MachineInstr *MI）`。这些函数允许目标数据包化器向现有数据包添加指令，并检查是否可以将指令添加到数据包中。有关更多信息 in `llvm/CodeGen/DFAPacketizer.h`。






