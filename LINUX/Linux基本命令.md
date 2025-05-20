# Linux 命令

## 基本命令

### 文件操作

#### 文件匹配

* `*`符号多字符匹配

* `?`符号单字符匹配

* glob`[]` 匹配

  * ``` bash
    hacker@dojo:~$ touch file_a
    hacker@dojo:~$ touch file_b
    hacker@dojo:~$ touch file_c
    hacker@dojo:~$ ls
    file_a	file_b	file_c
    hacker@dojo:~$ echo Look: file_[!ab]
    Look: file_c
    hacker@dojo:~$ echo Look: file_[^ab]
    Look: file_c
    hacker@dojo:~$ echo Look: file_[ab]
    Look: file_a file_b
    ```

```
```



### 管道符

####  `>`

覆盖文件所有内容。

```bash
echo "PWN" > a
# PWN
```

#### `>>`

将内容追加在文件后面。

```bash
echo "HELLO" > a
echo "PWN" >> a
# HELLO
# PWN
```

#### `|`

`|` 将程序输出，直接传给另一个程序

```bash
cat file | grep "pwn"
```

#### `>&`

重定向文件描述符

```bash
echo "abcd" 2>&1 | grep "b"
echo "abce" 2> 1.txt
```

####  >()

 ` >(cmd2)`生成一个临时管道文件（如 `/dev/fd/63`)

```bash
hacker@dojo:~$ echo >(rev)
/dev/fd/63 #bash 替换了 >(rev) 为指向 rev 输入端口的命名管道文件的路径！在命令运行期间，向这个文件写入数据将会将数据通过管道传递给该命令的标准输入。
hacker@dojo:~$ echo HACK | tee >(rev)
HACK
KCAH
hacker@dojo:~$ /challenge/hack 1> >(/challenge/planet) 2> >(/challenge/the)
```

#### cat

```bash
# 输入多行内容并保存
$ cat > output.lo
This is line 1.
This is line 2.
Ctrl+D  # 结束输入

# 查看文件内容
$ cat output.lo
This is line 1.
This is line 2.
```

###  alias 设置别名

```bash
alias[别名]=[指令名称]
```

#### 用法：

* 创建别名

    ```bash
    alias ll='ls -alF'
    ```

* 显示别名

  ```bash
  alias
  ```

* 删除别名

  ```bash
  unalias ll
  ```


### Shell Variables

#### 普通赋值

环境变量赋值，不能有空格

```bash
hacker@dojo:~$ VAR=1337 #只会在当前shell的本地变量中生效，其他程序无法继承。
hacker@dojo:~$ echo "VAR is: $VAR"
VAR is: 1337
hacker@dojo:~$ sh
$ echo "VAR is: $VAR"
VAR is: 
```

#### export

```bash
hacker@dojo:~$ VAR=1337 
hacker@dojo:~$ export VAR #使其能被其他程序使用。
hacker@dojo:~$ export PWN=12345
hacker@dojo:~$ sh
$ echo "VAR is: $VAR"
VAR is: 1337
$ echo "PWN is: $PWN"
PWN is: 12345 
```

#### env

 it'll print out every *exported* variable set in your shell.

```bash
hacker@dojo:~$ env
```

#### 将程序输出读入环境变量

```bash
hacker@dojo:~$ PWN=`cat /flag`
hacker@dojo:~$ PWN=$(cat /flag)
```

#### read

```bash
hacker@dojo:~$ echo "test" > some_file
hacker@dojo:~$ read VAR < some_file
hacker@dojo:~$ echo $VAR
test
```

#### 环境变量保存文件

* `/proc/self/environ ` 是一个虚拟文件，存储了当前进程的环境变量。

### 进程管理

#### ps

```bash
ps -ef # 查看详细信息
ps aux # 查看详细信息
ps -o user,pid,stat,cmd #自定义列表
```

  #### kill

```
kill
```

#### Ctrl + C

中断程序

#### Ctrl + Z

暂停程序

#### bg

- **恢复暂停的进程**：将已暂停的进程（如通过 `Ctrl+Z` 暂停的进程）转为后台运行。

- **释放终端**：进程在后台运行时，终端可继续输入其他命令，避免阻塞。

  ```bash
  hacker@processes~resuming-processes:~$ /challenge/run
  ^Z
  [1]+  Stopped                 /challenge/run
  hacker@processes~resuming-processes:~$ bg /challenge/run 
  /challenge/run 
  ```

#### fg

恢复暂停的程序，并恢复到前台

```bash
hacker@processes~resuming-processes:~$ /challenge/run
Let's practice resuming processes! Suspend me with Ctrl-Z, then resume me with 
the 'fg' command! Or just press Enter to quit me!
^Z
[1]+  Stopped                 /challenge/run
hacker@processes~resuming-processes:~$ fg /challenge/run 
/challenge/run 
I'm back! Here's your flag:
```

#### $?

查看程序是否运行成功

```bash
hacker@dojo:~$ touch test-file
hacker@dojo:~$ echo $?
0
hacker@dojo:~$ touch /test-file
touch: cannot touch '/test-file': Permission denied
hacker@dojo:~$ echo $?
1
hacker@dojo:~$
```

## 用户管理与权限

### su

### 密码破解

**密码位置**: /etc/shadow

**文件格式:**

```bash
root:$6$s74oZg/4.RnUvwo2$hRmCHZ9rxX56BbjnXcxa0MdOsW2moiW8qcAl/Aoc7NEuXl2DmJXPi3gLp7hmyloQvRhjXJ.wjqJ7PprVKLDtg/:19921:0:99999:7:::
daemon:*:19873:0:99999:7:::
bin:*:19873:0:99999:7:::
hacker::19916:0:99999:7:::
zardus:$6$bEFkpM0w/6J0n979$47ksu/JE5QK6hSeB7mmuvJyY05wVypMhMMnEPTIddNUb5R9KXgNTYRTm75VOu1oRLGLbAql3ylkVa5ExuPov1.:19921:0:99999:7:::
... ... 
```

每行中的第一个字段是用户名，第二个字段是密码。值为 * 或 ！ 。以 ：s 分隔。功能上讲，账户的密码登录已被禁用，一个空白字段意味着没有密码（这是一种常见但错误的配置，即在某些配置中允许无密码的 su 操作)。

使用John the Ripper工具破解hash

### 权限

#### chown

```
chown [username] [file]
```

#### chgrp 

```
chown [username] [file]
```

#### id

查看当前用户uid、所在组id

```
id
```

#### chmod

- `u+r`, as above, adds read access to the user's permissions
- `g+wx` adds write and execute access to the group's permissions
- `o-w` *removes* write access for other users
- `a-rwx` removes all permissions for the user, group, and world

##### 赋权限方式

* 数字

  * r - 4  w - 2  x - 1 

    ```bash
    chmod 444 [file]
    #所有者、组、其他人
    ```

* 字母

  * `o=-` `-`将 其他人权限设置为无

    ```bash
    chmod u=rw,g=r,o=- [file]
    ```

  * suid 设置

    * suid设置后程序将以文件所有者的权限运行

    ```bash
    chmod u+s [program]
    ```

    



## `namespace`

### `unshare`
* `-m`  unshare mounts namespace
* `-n`  unshare network namespace
* `-p --mount-proc --fork` unshare pid namespace
#### 实例
```bash
 sudo unshare -m -n -p --mount-proc --fork bash
```
### mount
挂载文件系统
```bash
mount --bind /usr ./root/usr 将/usr挂载到./root/usr
```
### umount 
取消挂载
```bash
umount ./root/usr 取消挂载
```
### pivot_root 
更改根目录
```bash
pivot_root [旧] [新]
```
## 调试
### `strace`

`strace`命令是一个集诊断、调试、统计与一体的工具，可用来追踪调试程序，能够与其他命令搭配使用

1、在操作系统运维中会出现程序或系统命令运行失败，通过报错和日志无法定位问题根因。

2、如何在没有内核或程序代码的情况下查看系统调用的过程。

## 网络请求
### `nc`

#### 发送`HTTP`请求

##### `GET`

````bash
nc 127.0.0.1 80
GET / HTTP/1.1
Host: exmple   #添加Host头
````

### `curl`

#### 发送`http`请求

* `-G` 发送GET请求，不加默认。

```
curl -G www.baiud.com
```

* `-H` 设置Host

  ```` 
  curl -H "Host: exp.com" url 
  ````

* `-X` 发送POST请求

#### 上传文件

```bash
curl -X POST -F xx=@flag.php  http://aaa # xx表单字段
```

### `man`

`man`命令用于查看系统手册页的工具。

```
man [选项] [命令符号]
```

#### 查找

使用  / 查找 字符串， n 用来查找下一个。N用来查找上一个。

### help

有些内置命令man手册中没有，所以使用help。

```help cd
help cd
```



