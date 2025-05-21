# GetLibc

### 使用`string`查看libc版本

```bash
strings ./libc.so.6 | grep "Ubuntu GLIBC"
```

### 使用ldd

```bash
ldd --version
```

