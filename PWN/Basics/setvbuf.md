# std缓冲区

### setvbuf

####　参数

- **stream** -- 这是指向 FILE 对象的指针，该 FILE 对象标识了一个打开的流。

- **buffer** -- 这是分配给用户的缓冲。如果设置为 NULL，该函数会自动分配一个指定大小的缓冲。

- **mode** -- 这指定了文件缓冲的模式：

  | 模式       | 描述                                                         |
  | :--------- | :----------------------------------------------------------- |
  | _IOFBF　０ | **全缓冲**：对于输出，数据在缓冲填满时被一次性写入。对于输入，缓冲会在请求输入且缓冲为空时被填充。 |
  | _IOLBF　１ | **行缓冲**：对于输出，数据在遇到换行符或者在缓冲填满时被写入，具体视情况而定。对于输入，缓冲会在请求输入且缓冲为空时被填充，直到遇到下一个换行符。 |
  | _IONBF　２ | **无缓冲**：不使用缓冲。每个 I/O 操作都被即时写入。buffer 和 size 参数被忽略。 |

- **size** --这是缓冲的大小，以字节为单位。

