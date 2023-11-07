# Arch-L4 Manual

#### 新内存模块重点

```verilog
parameter
	ADDR_WIDTH = 11;

localparam
	S_IDLE = 0,
	S_READING1 = 1,
	S_READING2 = 2,
	S_READ = 3,
	S_WRITING1 = 4,
	S_WRITING2 = 5,
	S_WRITE = 6;

always @ (*) begin
	if (cs) begin
		if (we) begin
			if(state == S_IDLE)
				next_state = S_WRITING1;
			else if (S_WRITING1 <= state && state <= S_WRITING2)
				next_state = state + 1;
			else if (state == S_WRITE)
				next_state = S_IDLE;
			else
				next_state = 3'bxxx;
		end
		else begin
			if (S_IDLE <= state && state <= S_READING2)
				next_state = state + 1;
			else if (state == S_READ)
				next_state = S_IDLE;
			else
				next_state = 3'bxxx;
		end
	end
	else begin
		next_state = S_IDLE;
	end
end
```

* 代码中无意义的读/写的多状态跳转是用于实际工作中内存的低速运行，从行为逻辑本身来说没有必要
* 在state == S\_READ或state == S\_WRITE时，ack信号设置为1

#### CMU

**CMU状态机**

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption><p>状态机一览</p></figcaption></figure>

* S\_IDLE：cache读写命中
* S\_PRE\_BACK：cache读写miss且cache当前槽位dirty，读取cache对应槽位，准备写入内存
* S\_BACK：上升沿将上个状态的数据写回到memory，下降沿从cache读下次需要写回的数据（因此最后一次读无意义），由计数器控制直到整个cache line全部写回，需要等待ack信号
* S\_FILL：上升沿从memory读取数据，下降沿向cache写入数据，由计数器控制直到整个cache line全部写入，需要等待ack信号
* S\_WAIT：执行之前由于miss而不能进行的cache操作

> 一旦cache miss，整个CPU就需要stall，等待miss之后对cache、内存的操作结束

**CMU cache控制信号**

```verilog
always @ (*) begin
    case(state)
        /* 外部指令读写cache */
        S_IDLE, S_WAIT: begin
            cache_addr = addr_rw;
            cache_load = en_r;
            cache_edit = en_w;
            cache_store = 1'b0;
            cache_u_b_h_w = u_b_h_w;
            cache_din = data_w;
        end
        /* cache dirty，执行write back by word */
        S_BACK, S_PRE_BACK: begin
            cache_addr = {addr_rw[ADDR_BITS-1:BLOCK_WIDTH], next_word_count, {ELEMENT_WORDS_WIDTH{1'b0}}};
            //                                                                | WORD_BYTES_WIDTH |
            /* cache不允许外部读写操作 */
            cache_load = 1'b0;  
            // cache.v中提到，load == 0即表示将cache内容发送到内存
            cache_edit = 1'b0;
            cache_store = 1'b0;
            cache_u_b_h_w = 3'b010; // write back by unsigned word
            cache_din = 32'b0;
        end
        /* cache clean，执行write allocate by word */
        S_FILL: begin
            cache_addr = {addr_rw[ADDR_BITS-1:BLOCK_WIDTH], word_count, {ELEMENT_WORDS_WIDTH{1'b0}}};
            //                                                                | WORD_BYTES_WIDTH |
            /* cache通过内存发出ack信号，决定是否从内存进行写操作 */
            cache_load = 1'b0;
            cache_edit = 1'b0;
            cache_store = mem_ack_i;
            cache_u_b_h_w = 3'b010; // write allocate by unsigned word
            cache_din = mem_data_i;
        end
    endcase
end
```

**CMU 内存控制信号**

```verilog
always @ (*) begin
    case (next_state)
        /* 不对内存操作 */
        S_IDLE, S_PRE_BACK, S_WAIT: begin
            mem_cs_o = 1'b0;
            mem_we_o = 1'b0;
            mem_addr_o = 32'b0;
        end
        /* 对内存写 */
        S_BACK: begin
            mem_cs_o = 1'b1;
            mem_we_o = 1'b1;
            mem_addr_o = {cache_tag, addr_rw[ADDR_BITS-TAG_BITS-1:BLOCK_WIDTH], next_word_count, {ELEMENT_WORDS_WIDTH{1'b0}}};
            //                                                                                    | WORD_BYTES_WIDTH |
        end
        /* 对内存读 */
        S_FILL: begin
            mem_cs_o = 1'b1;
            mem_we_o = 1'b0;
            mem_addr_o = {addr_rw[ADDR_BITS-1:BLOCK_WIDTH], next_word_count, {ELEMENT_WORDS_WIDTH{1'b0}}};
            //                                                                | WORD_BYTES_WIDTH |
        end
    endcase
end
```
