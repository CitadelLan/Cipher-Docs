# Arch-L6 Manual

#### Scoreboard

* 提出理由
  * 五级流水线虽好，但无论经过多少优化，我们能够达到的最优CPI也就是1，但我们或许可以做到更优的CPI。
  * 在进行运算的时候，不同的运算所消耗的时间是截然不同的。例如下图中，除法所消耗的时间是极长的，但下面的减法指令与上两条指令没有关系，完全可以独立先行运行掉。在五级流水线中，减法就需要等待上面的除法结束运算后才能运行，CPU的运行效率就被一条除法指令拖累了很多。&#x20;
  * 自然地，我们会考虑根据不同指令类型的运算时长与占比期望来拆分掉这些运算模块，并且这些模块可以是复数个，这样就可以帮助尽可能降低CPU因为上一点问题所带来的stall。&#x20;
* FU结构&#x20;

```verilog
// function unit
`define FU_BLANK    3'd0
`define FU_ALU      3'd1
`define FU_MEM      3'd2
`define FU_MUL      3'd3
`define FU_DIV      3'd4
`define FU_JUMP     3'd5

// bits in FUS
`define BUSY    0
`define OP_L    1
`define OP_H    5
`define DST_L   6
`define DST_H   10
`define SRC1_L  11
`define SRC1_H  15
`define SRC2_L  16
`define SRC2_H  20
/* FU1/2: SRC1/2 依赖的 FU */
`define FU1_L   21
`define FU1_H   23
`define FU2_L   24
`define FU2_H   26
`define RDY1    27
`define RDY2    28
`define FU_DONE 29
```

* 工作原理
  *   ISSUE：设置对应的FU，如果流水线陷入stall状态，那么对应指令会在这里重新给FU赋值

      ```verilog
      wire[4:0] src1 = {5{R_valid | I_valid | S_valid | L_valid | B_valid | JALR}} & rs1;
      wire[4:0] src2 = {5{R_valid | S_valid | B_valid}} & rs2;
      wire[2:0] fu1 = RRS[src1];
      wire[2:0] fu2 = RRS[src2];
      wire rdy1 = ~|fu1;
      wire rdy2 = ~|fu2;

      // IS
      if (RO_en) begin
      	// not busy, no WAW, write info to RRS and FUS
      	if (|dst) RRS[dst] <= use_FU;
      	FUS[use_FU][`BUSY]           <= 1'b1;
      	FUS[use_FU][`OP_H:`OP_L]     <= op;
      	FUS[use_FU][`DST_H:`DST_L]   <= dst;
      	FUS[use_FU][`SRC1_H:`SRC1_L] <= src1;
      	FUS[use_FU][`SRC2_H:`SRC2_L] <= src2;
      	FUS[use_FU][`FU1_H:`FU1_L]   <= fu1;
      	FUS[use_FU][`FU2_H:`FU2_L]   <= fu2;
      	FUS[use_FU][`RDY1]           <= rdy1;
      	FUS[use_FU][`RDY2]           <= rdy2;
      	FUS[use_FU][`FU_DONE]        <= 1'b0;

      	IMM[use_FU] <= imm;
      	PCR[use_FU] <= PC;
      end
      ```

      > RRS：记录WB阶段提供给对应寄存器覆写值 的 FU
  *   RO：等待两个输入值直到两者均为正确值当这两个元素均RDY之后，将两个RDY值置成0

      ```verilog
      // RO
      if (FUS[`FU_JUMP][`RDY1] & FUS[`FU_JUMP][`RDY2]) begin
      	// JUMP
      	FUS[`FU_JUMP][`RDY1] <= 1'b0;
      	FUS[`FU_JUMP][`RDY2] <= 1'b0;
      end
      ```

      > Scoreboard结构中，我们需要等待需要等待的寄存器的RDY信号为Yes/1时才可以继续，即进行stall。
  *   EX：等待对应模块运算结束，运算结束后对应 FU 的`FU_Done`信号设置为输入中对应的结束信号（ALU\_Done、MEM\_Done、...）

      > 对于目前空闲的FU模块，需要保持其原来的`FU_Done`
  *   WB 阶段的 WAR 检测

      ```verilog
      	wire ALU_WAR = (
      		(FUS[`FU_MEM ][`SRC1_H:`SRC1_L] != ... | ...)  & 
      		(FUS[`FU_MEM ][`SRC2_H:`SRC2_L] != ... | ...)  & 
      		(FUS[`FU_MUL ][`SRC1_H:`SRC1_L] != ... | ...)  & 
      		(FUS[`FU_MUL ][`SRC2_H:`SRC2_L] != ... | ...)  & 
      		(FUS[`FU_DIV ][`SRC1_H:`SRC1_L] != ... | ...)  & 
      		(FUS[`FU_DIV ][`SRC2_H:`SRC2_L] != ... | ...)  & 
      		(FUS[`FU_JUMP][`SRC1_H:`SRC1_L] != ... | ...)  & 
      		(FUS[`FU_JUMP][`SRC2_H:`SRC2_L] != ... | ...)    
      	);
      ```

      > 这里代码的逻辑是以 0 代表有 WAR，1 代表没有 WAR

      * 当前指令所处的 FU 模块所用到的**目标寄存器**与 上一指令所处的 FU 模块的**源寄存器**相同，即先读后写
      * 上一指令的**目标寄存器**准备好了（在RO阶段之后，RDY信号就会变为No/0）
      * 在本次实验中，WAR如果存在，且后发射指令（W指令）先到达WB会怎么样？ Scoreboard算法会在 WB 阶段检测后发射指令是否先进入WB，如果有的话则会对这条指令进行stall，直到先发射的指令WB完毕。从代码角度来看：

      ```verilog
      /* 时序逻辑中WB阶段 */
      // JUMP
      if (FUS[`FU_JUMP][`FU_DONE] & JUMP_WAR) begin
              FUS[`FU_JUMP] <= 32'b0;
              RRS[FUS[`FU_JUMP][`DST_H:`DST_L]] <= 3'b0;
      		......
      end

      /* 组合逻辑中WB阶段 */
      if (FUS[`FU_JUMP][`FU_DONE] & JUMP_WAR) begin
          write_sel = 3'd4;
          reg_write = 1'b1;
          rd_ctrl = FUS[`FU_JUMP][`DST_H:`DST_L];
      end
      ```

      这里可能还说的不是很明显，请回忆WAR变量的赋值逻辑：WAR赋值与对应FU的数据依赖关系和RDY位有关，而RDY的赋值主要是在ISSUE阶段进行。如果RDY为1，则上一条指令还在ISSUE阶段，否则其输入数据仍未准备好，或已经在后续流程中。一旦上一条指令进入了RO阶段，需要的输入和操作就已经确定，当前指令就可以放心写回了。

      ```verilog
      // JUMP
      if (FUS[`FU_JUMP][`RDY1] & FUS[`FU_JUMP][`RDY2]) begin
          ALU_en = 1'b0;
          MEM_en = 1'b0;
          MUL_en = 1'b0;
          DIV_en = 1'b0;
          JUMP_en = 1'b1;

          JUMP_op = FUS[`FU_JUMP][`OP_H:`OP_L];
          rs1_ctrl = FUS[`FU_JUMP][`SRC1_H:`SRC1_L];
          rs2_ctrl = FUS[`FU_JUMP][`SRC2_H:`SRC2_L];
          PC_ctrl = PCR[`FU_JUMP];
          imm_ctrl = IMM[`FU_JUMP];
      end
      ```
  *   WB阶段的RAWstall释放

      ```verilog
      if (FUS[`FU_ALU][`FU1_H:`FU1_L] == `FU_JUMP) ...;
      ```

      * 从原理上来说，我们需要覆写F2里的值，去除对应依赖当前写回值的FU里的Qj、Qk以及Rj、Rk。在本工程中只需要设置Rj、Rk，即RDY1、RDY2。

      > 那我们寄存器重命名后的对应存储位置的值怎么办呢？就个人目前对框架的理解，本实验中指令运算的输入值依然是通过GPR（通用寄存器）来取值的，所谓寄存器重命名只是明确指令流中的数据依赖关系，即：

      ```verilog
      FUS[`FU_ALU][`FU1_H:`FU1_L] == `FU_JUMP
      ```
