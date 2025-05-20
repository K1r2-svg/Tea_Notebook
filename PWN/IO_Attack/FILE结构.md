# FILE结构

### 代码结构

#### _IO_FILE_plus

```C
struct _IO_FILE_plus {
    _IO_FILE    file;      // 基础文件流结构
    _IO_jump_t *vtable;    // 虚函数表指针
};
```

#### _IO_FILE

FILE 结构定义在 libio.h 中

```C
struct _IO_FILE {
  int _flags;       /* 高位字是 _IO_MAGIC（固定值 0xfbad），低位是状态标志位 */
#define _IO_file_flags _flags  // 宏定义，简化对 _flags 的访问

  /* 以下指针遵循 C++ streambuf 协议，用于缓冲区管理 */
  char* _IO_read_ptr;   /* 当前读取指针（指向缓冲区中未读取的数据） */
  char* _IO_read_end;   /* 读缓冲区的结束位置 */
  char* _IO_read_base;  /* 读缓冲区的起始位置（包含回退区） */
  char* _IO_write_base; /* 写缓冲区的起始位置 */
  char* _IO_write_ptr;  /* 当前写入指针（指向缓冲区中未写入的数据） */
  char* _IO_write_end;  /* 写缓冲区的结束位置 */
  char* _IO_buf_base;   /* 缓冲区保留区的起始位置 */
  char* _IO_buf_end;    /* 缓冲区保留区的结束位置 */

  /* 以下字段用于回退（undo）和备份操作 */
  char *_IO_save_base;  /* 非当前读缓冲区的起始位置（用于备份） */
  char *_IO_backup_base;/* 回退区域的第一个有效字符指针 */
  char *_IO_save_end;   /* 非当前读缓冲区的结束位置 */

  struct _IO_marker *_markers;  /* 标记链表，用于流定位（如 fseek） */

  struct _IO_FILE *_chain;  /* 文件流链表指针，链接所有打开的 FILE 对象 */
  
  int _fileno;  /* 封装的文件描述符（如 stdout 为 1，stdin 为 0） */
#if 0
  int _blksize;  /* 旧版块大小字段（已弃用） */
#else
  int _flags2;   /* 扩展标志位，用于高版本兼容性 */
#endif

  _IO_off_t _old_offset; /* 旧版文件偏移量（因原 _offset 字段过小而替换） */

#define __HAVE_COLUMN  /* 临时定义：支持列号跟踪 */
  unsigned short _cur_column;  /* 当前输出列号（用于格式化输出） */
  signed char _vtable_offset;  /* 虚表偏移量（用于劫持虚函数） */
  char _shortbuf[1](@ref);           /* 短缓冲区（用于小数据快速存取） */

  _IO_lock_t *_lock;  /* 文件锁指针（保证多线程安全） */
#ifdef _IO_USE_OLD_IO_FILE  // 旧版 IO_FILE 兼容性定义
};
```

####  _IO_FILE_complete

扩展结构体 `_IO_FILE_complete`
在 `_IO_FILE `的基础上添加宽字符支持、64 位偏移量等字段，包括 `_mode` 和 `_offset`

系统中调用`_IO_FILE`结构体时，其实将其转换成了`_IO_FILE_complete`来进行访问，

低版本代码可能仅保留 `_IO_FILE` 的基础字段，而忽略扩展部分

```C
struct _IO_FILE_complete {
    struct _IO_FILE _file;  // 基础文件流结构体，包含缓冲区、标志位等核心字段[2,4,6,7](@ref)
#endif  // 结束旧版 _IO_FILE 结构的条件编译

// 以下字段仅在特定版本宏定义下生效（如 _G_IO_IO_FILE_VERSION == 0x20001）
#if defined _G_IO_IO_FILE_VERSION && _G_IO_IO_FILE_VERSION == 0x20001
    _IO_off64_t _offset;    // 64位文件偏移量（支持大文件操作）[6,7](@ref)

// 宽字符处理相关字段（当系统支持宽字符时生效）
# if defined _LIBC || defined _GLIBCPP_USE_WCHAR_T
    struct _IO_codecvt *_codecvt;       // 宽字符编码转换器指针
    struct _IO_wide_data *_wide_data;   // 宽字符缓冲区管理结构体指针
    struct _IO_FILE *_freeres_list;     // 释放资源链表指针（用于内部内存管理）
    void *_freeres_buf;                 // 释放资源缓冲区指针[6,7](@ref)
// 非宽字符系统的填充字段（保持内存对齐）
# else
    void *__pad1;   // 填充字段1（占位符，保证结构体对齐）
    void *__pad2;   // 填充字段2
    void *__pad3;   // 填充字段3
    void *__pad4;   // 填充字段4
# endif  // 结束宽字符支持的条件编译

    size_t __pad5;  // 填充字段5（兼容不同平台的内存对齐需求）
    int _mode;      // 文件模式标志位（如文本/二进制模式、读写权限等）[2,3,6,7](@ref)

    /* 确保结构体兼容性的保留字段 */
    char _unused2[15 * sizeof (int) - 4 * sizeof (void *) - sizeof (size_t)]; 
    // 该数组用于填充结构体空间，保证不同平台下结构体大小一致[6,7](@ref)
#endif  // 结束版本宏定义的条件编译
};
```



#### _IO_jump_t

```C
struct _IO_jump_t {
    JUMP_FIELD(size_t, __dummy);         // 占位字段
    JUMP_FIELD(size_t, __dummy2);        // 占位字段
    JUMP_FIELD(_IO_finish_t, __finish);  // 关闭文件流时调用
    JUMP_FIELD(_IO_overflow_t, __overflow); // 缓冲区溢出时触发（如 `exit()` 调用）
    JUMP_FIELD(_IO_underflow_t, __underflow); // 缓冲区不足时读取数据
    JUMP_FIELD(_IO_underflow_t, __uflow);    // 读取单个字符的底层实现
    JUMP_FIELD(_IO_pbackfail_t, __pbackfail); // 回退字符处理
    JUMP_FIELD(_IO_xsputn_t, __xsputn);  // 批量写入数据（被 `fwrite` 调用）
    JUMP_FIELD(_IO_xsgetn_t, __xsgetn);  // 批量读取数据（被 `fread` 调用）
    JUMP_FIELD(_IO_seekoff_t, __seekoff); // 调整文件指针位置（`fseek` 依赖）
    JUMP_FIELD(_IO_seekpos_t, __seekpos); // 绝对位置调整
    JUMP_FIELD(_IO_setbuf_t, __setbuf);   // 设置缓冲区
    JUMP_FIELD(_IO_sync_t, __sync);       // 同步文件流状态
    JUMP_FIELD(_IO_doallocate_t, __doallocate); // 动态分配缓冲区
    JUMP_FIELD(_IO_read_t, __read);       // 底层读取函数
    JUMP_FIELD(_IO_write_t, __write);     // 底层写入函数
    JUMP_FIELD(_IO_seek_t, __seek);       // 底层定位函数
    JUMP_FIELD(_IO_close_t, __close);     // 关闭文件描述符
    JUMP_FIELD(_IO_stat_t, __stat);       // 获取文件状态
    JUMP_FIELD(_IO_showmanyc_t, __showmanyc); // 检查剩余可读数据
    JUMP_FIELD(_IO_imbue_t, __imbue);     // 区域设置相关
#if 0
    // 旧版未实现的字段（已弃用）
    get_column;  
    set_column;
#endif
};
```



