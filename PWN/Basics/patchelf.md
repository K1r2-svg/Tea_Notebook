# patchelf使用

### 替换libc

```bash
patchelf --set-interpreter ./ld-2.23.so ./pwn
patchelf --replace-needed libc.so.6 ./libc-2.23-x64.so ./pwn
```

