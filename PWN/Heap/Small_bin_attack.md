# Small bin attack

### What's small bin?

* small bin个数：62个，每个small bin也是一个由对应free chunk组成的循环双链表。
* small bin采用FIFO(先入先出)算法。
* small bin size：第一个chunk大小为16字节，后续每个small bin中chunk的大小依次增加8字节，即最后一个small bin的chunk为16 + 62 * 8 = 512字节。
* 合并操作：相邻的free chunk需要进行合并操作，即合并成一个大的free chunk。具体操作见下文free(small chunk)介绍。
* malloc(small chunk)操作：类似于fast bins，最初所有的small bin都是空的，因此在对这些small bin完成初始化之前，即使用户请求的内存大小属于small chunk也不会交由small bin进行处理，而是交由unsorted bin处理，如果unsorted bin也不能处理的话，glibc malloc就依次遍历后续的所有bins，找出第一个满足要求的bin，如果所有的bin都不满足的话，就转而使用top chunk，如果top chunk大小不够，那么就扩充top chunk，这样就一定能满足需求了(还记得上一篇文章中在Top Chunk中留下的问题么？答案就在这里)。注意遍历后续bins以及之后的操作同样被large bin所使用。
* free(small chunk)：当释放small chunk的时候，先检查该chunk相邻的chunk是否为free，如果是的话就进行合并操作：将这些chunks合并成新的chunk，然后将它们从small bin中移除，最后将新的chunk添加到unsorted bin中。