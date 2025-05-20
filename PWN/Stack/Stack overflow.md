# Stack overflow

### Canary bypass

#### Leak Canary

##### 覆盖canary的\x00

##### 利用格式化字符串漏洞

##### 寻找遗留在stack上的canary
当输出函数如printf做了字符输出限制，可以查看输出范围内是否遗留了上个函数调用完未清理的canary。

### 栈迁移
#### exp
通过 `pop rbp ret`和`leave ret`实现栈迁移

### ROP

#### 寻找 pop ret指令

使用ROPgaget

#### 没有pop ret 至目标寄存器

寻找 mov ret add ret 等指令。

### 其他骚操作
#### 通过覆盖rbp执行存放在栈上的代码段中的代码。
** 利用前提 ：** 已知栈上地址。
将rbp覆盖成目标地址-8的位置，然后通过两次 `leave ret` 实现。当前函数执行完，会有一次。在覆盖返回地址为`leave ret`为第二次。
**注意：**执行到`leave ret`时，栈上内容会发生变化，可能会调用失败。