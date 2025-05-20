

# Memory

### malloc

#### malloc_chunk

```C
struct malloc_chunk {
    INTERNAL_SIZE_T prev_size; // 前一个 chunk 的大小（如果空闲）
    INTERNAL_SIZE_T size; // 当前 chunk 的大小，包括头部
    struct malloc_chunk* fd; // 指向下一个空闲 chunk
    struct malloc_chunk* bk; // 指向上一个空闲 chunk
    struct malloc_chunk* fd_nextsize; // 指向下一个不同大小的空闲 chunk
    struct malloc_chunk* bk_nextsize; // 指向上一个不同大小的空闲 chunk
};
```
* size 字段的二进制结构

| 位位置（低→高） |         名称         |                       描述                        |
| :-------------: | :------------------: | :-----------------------------------------------: |
|   **第 0 位**   |   `PREV_INUSE` (P)   |            前一个 chunk 是否在使用中。            |
|   **第 1 位**   |   `IS_MMAPPED` (M)   | 当前 chunk 是否由 `mmap` 直接分配（非堆区内存）。 |
|   **第 2 位**   | `NON_MAIN_ARENA` (A) |   当前 chunk 是否属于非主分配区（如线程本地堆）   |

### malloc_consolidate

glibc.2.39源码

```c
static void malloc_consolidate(mstate av)
{
  mfastbinptr*    fb;                 /* current fastbin being consolidated */
  mfastbinptr*    maxfb;              /* last fastbin (for loop control) */
  mchunkptr       p;                  /* current chunk being consolidated */
  mchunkptr       nextp;              /* next chunk to consolidate */
  mchunkptr       unsorted_bin;       /* bin header */
  mchunkptr       first_unsorted;     /* chunk to link to */

  /* These have same use as in free() */
  mchunkptr       nextchunk;
  INTERNAL_SIZE_T size;
  INTERNAL_SIZE_T nextsize;
  INTERNAL_SIZE_T prevsize;
  int             nextinuse;

  atomic_store_relaxed (&av->have_fastchunks, false);

  unsorted_bin = unsorted_chunks(av); //获取unsortbin 头

  /*
    Remove each chunk from fast bin and consolidate it, placing it
    then in unsorted bin. Among other reasons for doing this,
    placing in unsorted bin avoids needing to calculate actual bins
    until malloc is sure that chunks aren't immediately going to be
    reused anyway.
  */

  maxfb = &fastbin (av, NFASTBINS - 1);
  fb = &fastbin (av, 0);
  do {
    p = atomic_exchange_acquire (fb, NULL);
    if (p != 0) {
      do {
	{
	  if (__glibc_unlikely (misaligned_chunk (p)))
	    malloc_printerr ("malloc_consolidate(): "
			     "unaligned fastbin chunk detected");

	  unsigned int idx = fastbin_index (chunksize (p));
	  if ((&fastbin (av, idx)) != fb)
	    malloc_printerr ("malloc_consolidate(): invalid chunk size");
	}

	check_inuse_chunk(av, p);
	nextp = REVEAL_PTR (p->fd);

	/* Slightly streamlined version of consolidation code in free() */
	size = chunksize (p);
	nextchunk = chunk_at_offset(p, size);
	nextsize = chunksize(nextchunk);

	if (!prev_inuse(p)) {
	  prevsize = prev_size (p);
	  size += prevsize;
	  p = chunk_at_offset(p, -((long) prevsize));
	  if (__glibc_unlikely (chunksize(p) != prevsize))
	    malloc_printerr ("corrupted size vs. prev_size in fastbins");
	  unlink_chunk (av, p);
	}

	if (nextchunk != av->top) {
	  nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

	  if (!nextinuse) {
	    size += nextsize;
	    unlink_chunk (av, nextchunk);
	  } else
	    clear_inuse_bit_at_offset(nextchunk, 0);

	  first_unsorted = unsorted_bin->fd;
	  unsorted_bin->fd = p;
	  first_unsorted->bk = p;

	  if (!in_smallbin_range (size)) {
	    p->fd_nextsize = NULL;
	    p->bk_nextsize = NULL;
	  }

	  set_head(p, size | PREV_INUSE);
	  p->bk = unsorted_bin;
	  p->fd = first_unsorted;
	  set_foot(p, size);
	}

	else {
	  size += nextsize;
	  set_head(p, size | PREV_INUSE);
	  av->top = p;
	}

      } while ( (p = nextp) != 0);

    }
  } while (fb++ != maxfb);
}


static void *
_int_malloc (mstate av, size_t bytes)
{
 ... ...
  if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
    {
      ... ... 
    }

  if (in_smallbin_range (nb))
    {
      ... ...
    }
  else
    {
      idx = largebin_index (nb);
      if (atomic_load_relaxed (&av->have_fastchunks))
        malloc_consolidate (av); //chunk大小 > 0x410触发malloc_consolidate。
    }
```

#### 代码逻辑

当chunk大小 >= 0x410 触发 malloc_consolidate()

##### **1. 初始化**

- 标记 `have_fastchunks` 为 `false`：表示 fast bins 即将被清空。
- 获取 `unsorted_bin` 头指针：用于后续插入合并后的块。
- 计算 fast bins 的起始 (`fb`) 和结束 (`maxfb`)：遍历所有 fast bins。

##### **2. 遍历 fast bins**

- **外层循环**：逐个处理从 0 到 `NFASTBINS-1` 的 fast bin。
- **原子操作**：使用 `atomic_exchange_acquire` 清空当前 fast bin 的链表，避免多线程竞争。

##### **2.1 校验 chunk 合法性**

- **内存对齐检查**：防止攻击者伪造 chunk 元数据。
- **fast bin 索引验证**：确保 chunk 大小与当前 fast bin 匹配。

##### **2.2 合并前向空闲块**

- **检查 `prev_inuse` 位**：若为 0，表示前一个 chunk 空闲。
- **合并逻辑**：
  1. 计算前向 chunk 的位置 (`p = chunk_at_offset(p, -prevsize)`)
  2. 验证前向 chunk 的大小是否与 `prevsize` 一致（防御堆溢出）。
  3. 调用 `unlink_chunk` 从 bin 中移除前向 chunk。
  4. 更新当前 chunk 的 `size`。

##### **2.3 合并后向空闲块**

- **检查 next chunk 是否为 `top chunk`**：
  - 若是，直接合并到 `top chunk`。
  - 若否，检查 `nextinuse` 位：
    - 若空闲：合并 next chunk，调用 `unlink_chunk`。
    - 若在使用中：仅清除 next chunk 的 `prev_inuse` 位。

##### **2.4 插入 unsorted bin**

- **链接到 unsorted bin 头部**：
  1. 将合并后的块插入 `unsorted_bin->fd`。
  2. 更新 `first_unsorted->bk` 指向新块。
- **处理 large chunk**：若合并后的块大小超出 small bin 范围，清空 `fd_nextsize` 和 `bk_nextsize`。
- **设置元数据**：
  - `set_head(p, size | PREV_INUSE)`：标记当前块大小和 `prev_inuse` 位。
  - `set_foot(p, size)`：设置 next chunk 的 `prev_size`（仅限空闲块）。

##### **2.5 合并到 top chunk**

- 若 next chunk 是 `top chunk`，直接将合并后的块加入 `top`，并更新 `av->top` 指针。

### calloc分配

#### 特性

* calloc不会从fastbins和tcache bin中分配内存（即使请求大小匹配），因为fastbins的chunk可能未合并，无法保证连续空间。
* 分配时优先从smallbins/unsortedbins或top chunk切割。
* calloc从small bin 中拿出一个空闲chunk后，会将其余chunk放入tcache bin 中（直到填满），并且只会检测对第一个 chunk进行了完整性检查，后面的chunk的检查缺失，就造成了fake chunk。

## free



