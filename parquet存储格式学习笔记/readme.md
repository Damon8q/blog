# Parquet存储格式
Parquet格式是一种列式存储文件格式，支持嵌套的数据。 灵感来自于Google Dremel系统论文。

## parquet文件结构是怎么样的？
### 最外层结构
![image](parquet文件格式-最外层)

头尾是Magic Number，如"Par1"。 Footer Len表示文件Footer部分的长度。 
Footer包含了所有列的metadata的开始位置，以及其他一些统计信息。
RowGroup是行组的意思，代表一批行数据的内容，比如10000行，RowGroup内部又是按列存储的。

### Footer
![image](parquet文件格式-Footer)

Footer部分整体是通过ThriftCompactProtocol编码，因此读取的时候parquet文件的时候，会首先读取Footer部分的Meta信息，
找到自己感兴趣的Column Chunk，然后去顺序读取列的数据。

有一个特点是Footer的meta信息里面是包含了Schema信息的，因此parquet文件也被称为自描述的。

### RowGroups
![image](parquet文件格式-RowGroups)

一个RowGroup内部包含了若干个Column chunk，一个Column chunk内部由多个Page组成，
Page是读写的最小单位，Page内部还包含header和数据部分，header包含此page的元信息。
MetaData部分可包含索引和元信息。

## 从parquet格式中学到了什么？
1. RowGroup的设计，不止让列在一起，行也可以在一起，并且可降低key冗余。
2. 多层索引和元信息设计，不同层提供不同精度的索引，最外层索引粒度越粗，最里层索引粒度越细。并且每层都可以保留相应的统计信息，如预聚合等。
