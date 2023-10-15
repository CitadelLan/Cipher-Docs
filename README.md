---
description: 流水线CPU设计
---

# Arch-L1

### 大致模块拆解

1. 时钟

* clk\_diff：提供一个稳定的时钟（**本实验中频率为200MHz**）
  * IBUFGDS：一个连接时钟信号BUFG或DCM的专用的差分信号输入缓冲器。在IBUFGDS中，一个电平接口用两个独立的电平接口（I和IB）表示。**只有在I和IB接口值不一致的时候才会为输出赋值，这个输出值与I端口一致**。在I和IB接口值不一致的时候，输出不变。
  * \#(${codes})是为模块中的一些值赋值。
* my\_clk\_gen：从输出上来看应该是分频器（看着不像自己写的）
  * clk\_disp：25MHz
  * clk\_cpu：10MHz

2. 按钮

* btn\_scan：从按钮阵列上读取输入，内置防抖动
  * 从输入上来看，应该是有五列四行按钮（还没看板子）
  * 输出中1所在的位决定了这个按钮在板上的位置

> 第5列第4行按钮为`interrupt`按钮 第5列第1行按钮为`step`按钮

3. **显示模块（暂略）**
4. RV32core：RISC-V流水线CPU架构
5. **VGA调试（暂略）**

### RISC-V 32位 CPU架构

1. 参数

* 输入：
  * debug\_en：debug enable（SW\[0]）
  * debug\_step：debug step clock（btn\_step：第5列第1行按钮）
  * \[6:0] debug\_addr：debug address
  * clk：main clock（clk\_cpu：100MHz）
  * rst：synchronous reset
  * interrupter：interrupt source, **for future use**（SW\[12]）
* 输出：
  * \[31:0] debug\_data, // debug data

2. IF阶段

* REG\_PC：PC寄存器
* add\_IF：得到PC+4
* **mux\_IF：应该是选取给到ID阶段的PC地址（TODO）**
* inst\_rom：读取对应地址的 指令

3. ID阶段

*   reg\_IF\_ID

    ```verilog
     module    REG_IF_ID(input clk, //IF/ID Latch
    					input rst,  //流水寄存器使能
    					input Data_stall,  //数据竞争等待
    					input flush,       //控制竞争清除并等待
    					input [31:0] PCOUT,  //指令存储器指针
    					input [31:0] IR,     //指令存储器输出

    					output reg[31:0] IR_ID,      //取指锁存
    					output reg[31:0] PCurrent_ID   //当前存在指令地址
    				);

    ```

    * STALL时（**DATA HAZARD**）：输出保持原样
    * FLUSH时（**CTRL HAZARD**）：
      * IR\_ID修改为：0x00000013，即ADDI zero zero 0（NOP）
      * PCurrent\_ID修改为：PCOUT地址
*   ctrl：解析指令（ID主要内容）并给出相关的控制指令：

    ```verilog
    module CtrlUnit(
    	input[31:0] inst,
    	input cmp_res,
    	output Branch, ALUSrc_A, ALUSrc_B, DatatoReg, RegWrite, mem_w,
    		MIO, rs1use, rs2use,
    	output [1:0] hazard_optype,
    	output [2:0] ImmSel, cmp_ctrl,
    	output [3:0] ALUControl,
    	output JALR
    	);
    ```

    * Branch：是否跳转：包含了branch指令的判定与jal、jalr指令，这个操作其实不严谨（）
    * ALUSrc\_A/B：ALU输入选择信号
    * DatatoReg：是否为L型指令（是否将存储数据写入寄存器）
    * mem\_w：是否为S型指令（mem\_write）
    * MIO：是否为L/S型指令（Multiuse I/O？）
    * rs1/2use：是否在指令中使用到了rs1/2寄存器
    * hazard\_optype：hazard类型
    * ImmSel：指令拆解成立即数的格式
    * cmp\_ctrl：比较器控制信号（选择比较方式）
    * ALUControl：ALU计算方式
    * JALR：是否为JALR指令
* register：多了个Dubug\_addr，和以前的没啥区别
* imm\_gen：生成立即数
* mux\_forward\_A/B：选取实际操作中需要使用到的在ALU的A/B端口的数值
* jump指令地址获取：
  * mux\_branch\_ID：根据指令是否为JALR类型，选择当前PC（jal）/rs1（jalr）寄存器的地址
  * add\_branch\_ID：获取跳转地址
* hazard\_unit：集合了hazard探测与forwarding的模块。需要手搓，还在搓着

4. EX阶段

* reg\_ID\_EX：里面有个u\_b\_h\_w（取fun3所在的三个bit），最后会在MEM阶段的data\_ram模块里使用，ubhw，即unsigned、byte、half word、word，具体的作用会在MEM阶段解释
* mux\_A/B\_EXE：选择进入ALU的A/B输入端口的数据
* alu
* mux\_forward\_EXE：根据forward\_ctrl\_ls（还没看懂这是啥的简称）决定是否使用mem阶段取出的数据

5. MEM阶段

* reg\_MEM\_WB
* data\_ram：根据指令给出的存入的数据长度（b/h/w）来存入数值

6. WB阶段

* reg\_MEM\_WB
* mux\_WB
