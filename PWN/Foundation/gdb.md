# GDB

### 基本命令

#### b

#### info

#### c

#### r

### GDB 调试多进程

(1)调试父进程：set follow-fork-mode parent （缺省值，即默认）

(2)调试子进程：set follow-fork-mode child

(3)设置调试模式：set detach-on-fork [on | off] （缺省值on）

on：调试当前进程的时候，其它的进程继续运行。

off：调试当前进程的时候，其它的进程被gbd挂起。

(4)查看可调试的进程：info inferiors

(5)切换调试的进程：inferior 进程id


### GDB 调试多线程

(1)查看可切换调试的线程：info threads

(2)切换调试的线程：thread 线程id

(3)只运行当前线程：set scheduler-locking on

(4)运行全部的线程：set scheduler-locking off

(5)指定某线程执行某gdb命令：thread apply 线程id gdb_cmd

(6)全部的线程执行某gdb命令：thread apply all gdb_cmd

### 远程pwntools调用gdb调试

```python
context.terminal = ['tmux','splitw','-h']
```

### 查早environ

```
p &environ
```

