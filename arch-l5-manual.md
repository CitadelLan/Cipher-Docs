# Arch-L5 Manual

#### FU模块

> 在新CPU框架下，EX、MEM阶段被重新调整为进FU阶段。在FU阶段中，由于引入了乘除法这两个需要更长时钟周期的操作，所以对于不同操作需要使用不同的逻辑与时间。同时，考虑到乱序执行的要求，需要尽可能进行操作解耦，所以需要拆分成多个计算模块。

*   ALU模块

    * ALU支持操作：

    ```verilog
    wire[31:0] res_ADD  = A + B;
    wire[31:0] res_SUB  = A - B;
    wire[31:0] res_AND  = A & B;
    wire[31:0] res_OR   = A | B;
    wire[31:0] res_XOR  = A ^ B;
    wire[31:0] res_SLL  = A << shamt;
    wire[31:0] res_SRL  = A >> shamt;

    wire[31:0] res_SLT  = {31'b0, res_SUB[31] ^ sub_of};
    wire[31:0] res_SLTU = {31'b0, res_subu[32]};
    wire[31:0] res_SRA  = {{32{A[31]}}, A} >> shamt;
    wire[31:0] res_Ap4  = A + 4;
    wire[31:0] res_Bout = B;
    ```

    * 判断ALU模块计算完毕：使用一个二元状态的状态机以表示当前模块是否空闲，但是这一状态机在与时钟同步的情况下，应该会产生一个时钟周期的浪费，对性能有负面影响

    ```verilog
    always@(posedge clk) begin
        /* state == 0，ALU available，填参数 */
        if(EN & ~state) begin
            A <= ALUA;
            B <= ALUB;
            Control <= ALUControl;
            state <= 1;
        end
        /* state == 1，ALU unavailable，重置状态 */
        else state <= 0;
    end
    ```
* 乘法器
  * 状态判断：思路类似于ALU，且由于乘法需要的时钟周期较长，使用一个7状态的状态机以用于等待。状态机为一个 7 bit 的变量，第 n 位为 1 即表示当前状态处于第 n 状态。在状态机达到0状态时，输出结束信号，即`assign finish = state[0] == 1'b1`
* 除法器
  * 状态判断：类似于ALU的状态机。由于除法需要的时钟周期非常多，如果与乘法器一样使用多重状态机的思路会显得过于麻烦，所以通过下属除法器模块自身，输出除法结束的标志。两者都结束即`assign finish = res_valid & state`
* 内存操作
  * 状态判断：思路类似于乘法器，2 bit状态机
* 跳转操作&#x20;
  *   在跳转指令进入FU阶段时，使能信号EN会被激活，这个时候的`state==0`，所以会去执行：

      ```verilog
      	JALR_reg <= JALR;
      	cmp_ctrl_reg <= cmp_ctrl;
      	rs1_data_reg <= rs1_data;
      	rs2_data_reg <= rs2_data;
      	imm_reg <= imm;
      	PC_reg <= PC;
      	state <= 1;
      ```
  * 在这个赋值中state被赋为1，因为`finish = state == 1‘b1`，所以这个时候finish信号就被置位了
  * 比较器进行两个rs的比较，两个加法器分别计算出跳转以及不跳转的指令地址，CtrlUnit根据比较器的结果来选择是进入跳转指令的地址还是不跳转指令的地址

#### 四级流水线流程

**Hazard handling**

* 无Forwarding、Hazard detection
* 在当前 FU、WB 阶段结束之后，才允许下一条指令从 IF 阶段进入到 FU 阶段。该方法可以有效避免data hazard，但对没有data hazard的指令而言，无脑stall会降低其执行效率

> 就代码层面而言，在WB阶段对应运算的寄存器会输出`FU_xxx_finish`信号，这一信号会在`Ctrl_Unit`中接收并stall所有IF、ID阶段

**Branching**

* 使用`Predict not taken`，其优势在Lab1中已经阐述过，这里不再赘述
* `Branching`操作在 FU 阶段执行，如果预测错误，则flush掉branch后两条指令
