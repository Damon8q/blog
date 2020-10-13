# LevelDB学习笔记

## 基本原理和整体架构是怎样的?
整体存储结构是采用LSM结构，如下图所示：
![image](levedb.png)

1. MemTable：内存数据结构，具体实现是 SkipList。 接受用户的读写请求，新的数据会先在这里写入。
2. Immutable MemTable：当 MemTable 的大小达到设定的阈值后，会被转换成 Immutable MemTable，只接受读操作，不再接受写操作，然后由后台线程 flush 到磁盘上 —— 这个过程称为 minor compaction。
3. Log：数据写入 MemTable 之前会先写日志，用于防止宕机导致 MemTable 的数据丢失。 一个日志文件对应到一个 MemTable。
4. SSTable：Sorted String Table。分为 level-0 到 level-n 多层，每一层包含多个 SSTable，文件内数据有序。除了 level-0 之外，每一层内部的 SSTable 的 key 范围都不相交。
5. Manifest：Manifest 文件中记录 SSTable 在不同 level 的信息，包括每一层由哪些 SSTable，每个 SSTable 的文件大小、最大 key、最小 key 等信息。
6. Current：重启时，LevelDB 会重新生成 Manifest，所以 Manifest 文件可能同时存在多个，Current 记录的是当前使用的 Manifest 文件名。
7. TableCache：TableCache 用于缓存 SSTable 的文件描述符、索引和 filter。
8. BlockCache：SSTable 的数据是被组织成一个个 block。 BlockCache 用于缓存这些 block（解压后）的数据。

## MemTable 的内部编码是怎么样的?
![image](memtable.png)

MemTable 中保存的数据是 key 和 value 编码成的一个字符串，由四个部分组成：

klength: 变长的 32 位整数（varint 的编码），表示 internal key 的长度。
internal key: 长度为 klength 的字符串。
vlength: 变长的 32 位整数，表示 value 的长度。
value: 长度为 vlength 的字符串。因为 value 没有参与排序，所以相关的讨论暂时可以忽略。

MemTable 的 KeyComparator 负责从 memkey 中提取出 internalkey，最终排序逻辑是使用 InternalKeyComparator 进行比较，排序规则如下：

1. 优先按照 user key 进行排序。
2. User key 相同的按照 seq 降序排序。
3. User key 和 seq 相同的按照 type 降序排序（逻辑上不会达到这一步，因为一个 LevelDB 的 sequence 是单调递增的）。

所以，在一个 MemTable 中，相同的 user key 的多个版本，新的排在前面，旧的排在后面。

## Log的格式编码是怎么样的?
![image](log.png)

如上图所示，LevelDB 的 log 文件内容被组织成多个 32 KB 的定长块（block）。每个 block 由 1~多个 record 组成（末尾可能会 padding）。
一个 record 由一个固定 7 字节的 header（checksum: uint32 + length: uint16 + type: uint8） 和实际数据（data: uint8[length]）组成。

如果 block 的末尾不足 7 字节（小于 header 的大小），则全部填 0x00，读取的时候会被忽略。

下面，我们将上层写入的数据称之为 user record，以区分 block 中的 record。
由于 block 是定长的，而 user record 是变长的，一个 user record 有可能被截断成多个 record，保存到一段连续的 block 中。
因此，在 header 中有一个 type 字段用来表示 record 的类型：
```
enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
```
1. kFullType - 这是一个完整的 user record。
2. kFirstType -  这是 user record 的第一个 record。
3. kMiddleType - 这是 user record 中间的 record。如果写入的数据比较大，kMiddleType 的 record 可能有多个。
4. kLastType - 这是 user record 的最后一个 record。

## 为什么采用定长块的保存方式?
当日志文件发生数据损坏的时候，这种定长块的模式可以很简单地跳过有问题的块，而不会导致局部的错误影响到整个文件。


















