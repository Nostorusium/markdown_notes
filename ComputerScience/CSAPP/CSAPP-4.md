# CSAPP章节小结

## 第四章 处理器体系结构

硬件设计背景，描述处理器的基本结构块与如何连接与操作
给出Y86-64处理器 顺序操作
创建一个流水线化的处理器

### **逻辑设计(非重点)**

HCL与C语言逻辑表达式区别:
C语言中存在部分求值，组合逻辑中不存在。

考虑 (a&&!a) && func(b,c)
第一部分为 0 , 0 &&任何数都是 0 ,所以后一部分不会被求值
即便后一部分的func做了非法操作，也不会报错

组合逻辑电路不存储任何信息

### **存储器、时钟**

时钟：周期性信号，决定什么时候把新值加载到设备
两类存储器设备：
**时钟寄存器**：简称寄存器
**随机访问存储器**：简称内存

**硬件寄存器**：时钟上升沿才改变输出
**寄存器文件**：是随机访问存储器，写入受到时钟信号控制(高电平)

寄存器文件以寄存器标识符作为地址 它在逻辑上是一个组合逻辑块
当src给如某个寄存器的ID 那么val就会输出指定寄存器的内容
写入时同理 但写入需要受到时钟的控制

(判断题)
向存储器写入值受到时钟控制，存在时序；
从存储器读取值不受时间控制，不存在时序，近似于逻辑电路。

### **Y86-64指令集体系结构**

1. 程序员可见状态：
   程序寄存器、条件码(CC)、程序计数器(PC)、状态码(Stat)、内存

2. 指令编码 P246
   有的指令编码后、还需要**操作数**(指定某寄存器)、**地址**信息；
   当指明不访问任何寄存器时，ID值设为 **0xF** (选择题)

3. RISC与CISC(选择题)
   R:Reduced C:Complex
   Intel属于CISC，现在的手机一般是RISC  

### **Y86-64的顺序实现**

取指fetch 译码decode 执行execute 访存memory 写回write back 更新PC PC update

1. 分6个阶段(选择)
   取指：取icode,ifun,ra,rb,与可能存在的**valC**，计算**valP**
   译码：读入操作数，得到**valA,valB**.可能从寄存器读，也可能从%rsp读
   执行：ALU执行操作，可能设置条件码，修改栈指针 得到的值为**valE**；该阶段不再从寄存器读取，使用已有的valA/valB/valC计算；执行Op指令后更新CC(set CC)
   访存：写入内存或者读出数据，得到**valM**
   写回：最多写回两个结果到寄存器文件(用valE与valM写回)
   更新PC：PC <-- **valP**

   *流水线：只有5个阶段，对顺序进行了优化*

2. **写微指令(大题)** P266~P270
   >SEQ中各val的产生位置
   >取指Fetch:产生valP,valC
   >译码Decode:产生valA,valB;**valB只参与ALU运算**
   >执行Execute:使用valA,valB,valC,产生valE,set CC;
   >访存Memory:使用valA,valE,产生valM
   >写回WriteBack:使用valM,valE
   >更新PC:PC Update:使用valM,valC,valP

   *x86惯例：先访存，再更新%rsp*
   2022-854真题:写出pushq,popq
   Op,movq: P266
   pushq,popq: P268
   jump,call,ret: P270
   要点：valA,valB,valM能算提前算好，充分利用硬件
   设计新指令(目前没出过)

3. **SEQ硬件结构**
   指令**不允许回读**，即不会读取由该指令更新过的状态；

   需要时序：PC、CC、数据内存、寄存器文件
   不需要时序：组合逻辑、指令内存

   因为指令内存部分只读不写，所以不需要时序；

   P276　图 :真题图

### **流水线**

1. **流水线的通用原理**：
   在组合逻辑之间插入流水线寄存器，通过时序控制各阶段的操作，实现了流水，代价是多插入的几个寄存器带来的轻微延迟。

   **2021-854原题**：解释大题，结合示例
   >参照Y86-64流水线CPU的实现，说明流水线如何工作：
   >待执行的任务被划分为若干个独立的阶段，处理器的硬件被组织成若干个单元，让各个独立的任务阶段在不同的硬件单元上一次执行，从而使多个任务并行操作。Y86-64将指令分为取指，译码，执行，访存，写回五个阶段，在每个阶段插入流水线寄存器，利用时钟信号控制流水线的时序和操作，实现并行。
   >解释2分，结合示例3分

   局限性：
   1. 划分不一致：速率由最慢的阶段决定
   2. 流水线过深：划多了，寄存器带来的延迟变多，而性能却没有更大的提升，收益反而下降

2. **Y86-64的流水线实现** P291
   改动：把PC的更新放在取指阶段前面，即SEQ+ 让后续进来的并行任务知道自己从哪开始
   先提前算好，存储进pIcode与pCnd;PC存储前一个周期中算好的信号。
   插入流水线寄存器：PIPE-
   实现转发逻辑：PIPE：可以转发valP,valE,valM到valA,valB

   **分支预测**：
   **1. 预测下一条PC**
      要完全流水化，就要每个时钟周期都发射一条新的指令。
      而如条件跳转，PC跳转值可能为valP(PC+9,下一条指令),也可能为valC(M8[PC+1])，取决于执行阶段产生的Cnd信号
      再如ret，PC的返回值必须等到Memory阶段后才知道。(返回值在栈中，出栈)
      我们总是预测选择了条件分支:即预测PC值为valC
      而如ret，可能的返回值几乎是无限的。我们不做预测，只简单地暂停新指令，直到ret写回(更新栈指针)。

      在F流水线寄存器中，Predict PC(PredPC)来选择valP还是valC

   **2. 流水线冒险**
      **数据相关**：下一条指令需要上一条指令的计算结果；数据冒险
      **控制相关**：上一条指令需要确定下一条指令的位置；控制冒险
      1. **暂停**避免*数据冒险*：
         暂停某个阶段的执行；如，想迟滞在D，就往E插气泡，E就会暂停。
         插入bubble阻塞阶段，等一个或几个阶段再继续
         气泡相当于一个non指令,可以在任意某个阶段插,来阻塞这一阶段后的指令
      2. **转发**避免*数据冒险*
         修改一下硬件结构，允许上一条产生的valE作为当前译码阶段的valB
         无需等待写回W结束，D就可以正确的读到计算结果。
         PIPE的valA允许转发**valP**;valB允许转发**valE,valM**
      3. **加载/使用***数据冒险*
         转发只能向后转发，不能向前转发。
         如：当前指令需要的数据还未被上一条数据计算出。
         无法仅通过转发解决，必须暂停之后再转发。
         只等待**1**个周期
      4. 避免**控制**冒险
         控制冒险发生在ret与跳转指令。
         对于**ret指令**：没法在一开头就算出PC 只能等写回前的访存结束送出valM(栈中拿出返回地址)
         等待**3**个周期，直到ret计算出valM并进入写回阶段，再开始下一条指令的取指
         对于**跳转指令分支预测错误**：
         当条件操作进行到E，进行了条件码的设置后，条件分支发现预测错误。 P306
         此时，接下来执行的两条错误的指令都没有抵达执行阶段，没有改变条件码，故没有改变程序员可见状态。
         通过插入两个气泡，并取出跳转指令后面正确的指令，来取消两条错误的指令。
         浪费了**两个**时钟周期。
         这种向D与E插入气泡的方式可以简单的将两条错误的指令排除
         正确的指令一旦进入F 则后续的D与E就将舍弃原来2条错误指令的计算转而进行新的计算
