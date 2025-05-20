### 接受信息

#### recv()

```python
p.recv(100) #接受100个字节
```

#### recvuntil()

```python
p.recvuntil("hello",drop=True) #直到接受到hello，drop不保留hello
```

#### recvline()

```python
p.recvline(keepends=False)  #接受一行不保留换行符
```

#### recvall()

```python
p.recvall() #会持续读取目标程序的输出，直到程序结束或连接关闭
```

### 打开 gdb 调试

#### gdb.debug()

```python
p=gdb.debug("pwn","break main") #打断电不像 attach那样打的断点会过
```

#### 传gdb命令的方法

```python
gdbscript = '''
	b main
	b *0x3231
	c
'''
p = gdb.debug('./pwn',gdbscript)
gdb.attach(p,gdbscript)
```

### ELF

```python
elf = ELF('./pwn')

libc = elf.libc #获取程序libc信息

elf.got['puts'] #获取got偏移

elf.plt['puts'] #获取plt偏移

libc.sym['system'] #获取函数偏移

libc.search('/bin/sh').__next__() #获取字符串偏移

libc.search('pop rdi; ret').__next__()
```

### Kernel Pwn 文件上传脚本
