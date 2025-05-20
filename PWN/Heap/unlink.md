# Unlink

### unlink 源码

```C
static void
unlink_chunk (mstate av, mchunkptr p)
{
  if (chunksize (p) != prev_size (next_chunk (p)))
    malloc_printerr ("corrupted size vs. prev_size");

  mchunkptr fd = p->fd;
  mchunkptr bk = p->bk;

  if (__builtin_expect (fd->bk != p || bk->fd != p, 0))
    malloc_printerr ("corrupted double-linked list");

  fd->bk = bk;
  bk->fd = fd;
  if (!in_smallbin_range (chunksize_nomask (p)) && p->fd_nextsize != NULL)
    {
      if (p->fd_nextsize->bk_nextsize != p
	  || p->bk_nextsize->fd_nextsize != p)
	malloc_printerr ("corrupted double-linked list (not small)");

      if (fd->fd_nextsize == NULL)
	{
	  if (p->fd_nextsize == p)
	    fd->fd_nextsize = fd->bk_nextsize = fd;
	  else
	    {
	      fd->fd_nextsize = p->fd_nextsize;
	      fd->bk_nextsize = p->bk_nextsize;
	      p->fd_nextsize->bk_nextsize = fd;
	      p->bk_nextsize->fd_nextsize = fd;
	    }
	}
      else
	{
	  p->fd_nextsize->bk_nextsize = p->bk_nextsize;
	  p->bk_nextsize->fd_nextsize = p->fd_nextsize;
	}
    }
}
```

### unlink检测机制

```C
// 由于 P 已经在双向链表中，所以有两个地方记录其大小，所以检查一下其大小是否一致(size检查)
if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");               \
// 检查 fd 和 bk 指针(双向链表完整性检查)
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
  malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \
  // largebin 中 next_size 双向链表完整性检查 
              if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)              \
                || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    \
              malloc_printerr (check_action,                                      \
                               "corrupted double-linked list (not small)",    \
                               P, AV); 
```

#### unlink 攻击

现在有两个物理相邻的 chunk 分别是 Q 和 P

在Q 中构造  fake chunk ，绕过检测需要构造FD和BK 如： FD = &P - 0x18 ，BK = &P - 0x10

再伪造P的prev_size 为 fake chunk 的大小，并将其 prev_inuse 位清空。

free P 触发 unlink，

最后会向 &P 中 填入 &P - 0x18的地址。

#### 绕过检测

* 绕过`if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0)) `

  1.伪造的 fakechunk的 next chunk的prev_size 为 fake chunk 的大小，并将其 prev_inuse 位清空

* 绕过`if (__builtin_expect (FD->bk != P || BK->fd != P, 0)) `

  1. 我们需要一个fake_chunk。
  2. 制造 FD -> bk  和 BK -> fd 为 P的地址。
  3. 因此 FD 和 BK 可以找存放P指针的内存空间就是 &P。
  4. FD = &P - 0x18 ，BK = &P - 0x10。



