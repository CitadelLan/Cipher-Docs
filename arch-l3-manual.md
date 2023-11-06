# Arch-L3 Manual

#### Cache结构 / inner\_xxx

* inner\_recent：对应地址下是否是最近添加的元素（配合LRU）
* inner\_valid：对应地址下是否是合法元素
* inner\_dirty：对应地址下是否是脏元素（没写入内存）
* inner\_tag（23 bits / index）：对应地址的Tag
* inner\_data（32 bits / index）：对应地址的存储数据

> 需要指出，这里的`[0:ELEMENT_NUM*ELEMENT_WORDS-1]`实际上代表了这个cache一共64个槽位，每个槽位可以代表一个 4 word的地址空间（addr\[3:2] 可以表示4个连续word的地址空间），即cache实际上可以代表`[0:ELEMENT_NUM*ELEMENT_WORDS-1]`大小的地址空间。但这里有个很奇怪的地方是，实际上我们运用到的只有其中64个下标（即实际槽位数量）没有block ofs（cache它甚至不愿意\*4）

#### 地址解析

```
| ----------- address 32 ----------- |
| 31   9 | 8     4 | 3    2 | 1    0 |
| tag 23 | index 5 | word 2 | byte 2 |
```

#### Cache操作

* load / read：根据是否hit决定是否要读cache里的内容，并根据u\_b\_h\_w\[2:0]决定选取大小，但只读从基地址读
* 设置inner\_recent

> u\_b\_h\_w：（指定位为1即表示符合要求） 2：是否是有符号数（对于非word有需要） 1：是否是word（第一优先级） 0：是否是half\_word（第二优先级）

* edit / write into cache：
  * 原理类似于load / read，但是这里对于非word类型的写入位置是严格根据传入地址操作的，例如对于写入half\_word，需要知道在对应地址word的前一半还是后一半进行写操作，并且会保留原word的没有被覆盖写的另一半
  * 设置inner\_recent、inner\_dirty
* store / write into RAM
  * 根据LRU原则，替换掉最久没被调用的槽位上的数据，如果一个set里两个槽位都没有recent标记，则默认放在第一个槽位。
* invalid
  * 类似于rst，把inner\_recent / inner\_valid / inner\_dirty全部置零。
