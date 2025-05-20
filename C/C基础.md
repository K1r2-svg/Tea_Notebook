# C语言基础

### 函数使用

#### `__attribute__((constructor))`

`__attribute__((constructor))` 是GCC/Clang编译器的扩展语法，用于标记函数在**程序启动时自动执行**（早于`main()`函数）

示例中的 `nightmare()` 函数将在程序入口（如`main()`）前被调用，无需显式触发

```c
void __attribute__((constructor)) nightmare() {
    printf("Nightmare function runs before main!\n");
}
//可指定优先级数字（如constructor(200)），数值越小执行越早。默认未指定优先级的函数按编译顺序执行
__attribute__((constructor(101))) void init1() { ... }
__attribute__((constructor(102))) void init2() { ... }
```

#### alarm函数

`alarm` 函数是 C 标准库中的一个函数，用于在指定的时间后发送 `SIGALRM` 信号给调用进程。此函数定义在 <unistd.h> 头文件中。

```
unsigned int alarm(unsigned int seconds);
```

#### signal函数设置 SIGALRM 的信号处理程序

**SIGALRM信号**：是Linux系统中用于实现定时功能的信号。它通常用于在程序中设置一个闹钟，在达到指定时间后执行特定的操作。

改信号作用：例如，在socket编程中，如果客户端长时间没有与服务器交互，服务器可能需要在一定时间后主动关闭socket连接。这时，就可以利用SIGALRM信号来实现这一功能。

实例：

```C
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

// 信号处理程序
void handle_sigalrm(int sig) {
    printf("Caught signal %d: Alarm triggered\n", sig);
}

int main() {
    // 设置 SIGALRM 的信号处理程序
    signal(SIGALRM, handle_sigalrm);

    // 设置闹钟，在 5 秒后触发 SIGALRM 信号
    alarm(5);
    printf("Alarm set for 5 seconds\n");

    // 无限循环，等待信号
    while (1) {
        printf("Running...\n");
        sleep(1);
    }

    return 0;
}
```

### 文件操作

#### lseek()

##### 简述

用于改变文件描述符的文件偏移量。文件偏移量是文件中下一个读或写操作将要发生的位置。通过 lseek()，你可以将文件指针移动到文件的特定位置，从而实现对文件的随机访问。

##### 用法

```c
#include <unistd.h>
#include <sys/types.h>

off_t lseek(int fd, off_t offset, int whence);
```

###### 参数说明

- **`fd`**：文件描述符，表示需要操作的文件。
- **`offset`**：偏移量，表示文件指针需要移动的字节数。可以是正数（向前移动）或负数（向后移动）。
- **`whence`**：指定偏移量的参考位置，可以是以下值之一：
  - `SEEK_SET`：从文件开头开始计算偏移量。
  - `SEEK_CUR`：从当前文件指针位置开始计算偏移量。
  - `SEEK_END`：从文件末尾开始计算偏移量。

###### 返回值

- 成功时，返回新的文件偏移量（从文件开头计算的字节数）。
- 失败时，返回 `(off_t) -1`，并设置 `errno` 以指示错误类型。

###### 常见错误

- `EBADF`：文件描述符无效。

- `EINVAL`：`whence` 参数无效，或者文件偏移量超出了文件范围。

- `ESPIPE`：文件描述符指向的是一个管道、FIFO 或套接字，这些文件类型不支持 `lseek()`。

  

##### 实例

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 将文件指针移动到第 10 个字节
    off_t offset = lseek(fd, 10, SEEK_SET);
    if (offset == (off_t) -1) {
        perror("lseek");
        close(fd);
        return 1;
    }

    char buffer[100];
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer) - 1);
    if (bytes_read == -1) {
        perror("read");
        close(fd);
        return 1;
    }

    buffer[bytes_read] = '\0'; // 添加字符串结束符
    printf("从偏移量 10 开始读取的内容: %s\n", buffer);

    close(fd);
    return 0;
}
```

### SandBox编写

#### 所需函数

##### `mkstemp`

##### `chroot`

##### `chdir`

```
```
### socket 网络编程
