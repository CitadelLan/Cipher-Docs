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
  * WB
