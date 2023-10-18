# Arch-L2 Manual

### 新增模块

* CSRRegs（Control Status Registers）：在特权模式下的一组特殊的寄存器，用于修改硬件线程的行为或者状态，例如CPU执行状态、类型、trap handler地址。
  * Input：
    * `clk, rst`
    * `[11:0] raddr, waddr`：读写地址
    * `[31:0] wdata`：读数据
    * `csr_w`：写使能信号
    * `[1:0] csr_wsc_mode`：写模式
      * 00、01：在对应寄存器写入待写入数据
      * 10：在对应寄存器原有数据基础上 位或 待写入数据
      * 11：在对应寄存器原有数据基础上 位与 待写入数据取反的结果
  * Local Vars：
    * `reg[31:0] CSR [0:15]`：模块内部的CSR寄存器
    *   地址相关，以读地址为例（写地址判断赋值语句类似）：

        * `wire raddr_valid`：检测是否是合法的读地址，要求`raddr[11:7] == 5'h6 && raddr[5:3] == 3'h0`。对应到地址表里即意味着：寄存器可读写，当前模式为`Machine`模式。
        * `wire[3:0] raddr_map` ：读地址映射，值为`(raddr[6] << 3) + raddr[2:0]`。在本次实验中需要用到的寄存器的为：

        | raddr\[6] | raddr\[2:0] |  CSR名称  |                                        用途                                       |                            主要变量                           |
        | :-------: | :---------: | :-----: | :-----------------------------------------------------------------------------: | :-------------------------------------------------------: |
        |     0     |     000     | mstatus | 进入 trap 之前，把异常发生前 CPU 的状态信息保存到 mstatus 中。退出 trap 时，通过 mstatus 的存储值来恢复 CPU 的状态信息 |    <p>MPP：trap前权限<br>MIE：中断使能信号<br>MPIE：trap前中断使能信号</p>   |
        |     0     |     101     |  mtvec  |                               存储异常处理跳转到 trap 的基地址                               | <p>Base：trap基地址<br>Mode：跳转方式（本实验为Direct，即所有异常均跳转到基地址）</p> |
        |     1     |     001     |   mepc  |                           记录发生异常/中断的指令地址，分别为当前PC和下一PC                           |                            mepc                           |
        |     1     |     010     |  mcause |                                 记录异常/中断发生的原因和类型                                 |             <p>Interrupt<br>Exception Code</p>            |
        |     1     |     011     |  mtval  |           保存了 trap 的附加信息：page fault中出错的地址、发生非法指令例外的指令本身，对于其他异常，它的值为 0           |                             /                             |
        |     1     |     100     |   mip   |                              列出目前正准备处理的中断（已经到来的中断）                              |                             /                             |
  * Output：
    * \[31:0] rdata：CSR\[raddr\_map]
    * \[31:0] mstatus：CSR\[0]

### CSR指令

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption><p>CSR指令一览</p></figcaption></figure>

* CSRRW：`CSRRW rd, csr, rs1` <==> GPR\[rd] = CSR\[csr], CSR\[csr] = GPR\[rs1]
* CSRRS：

