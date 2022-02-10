## Introduction
> This document defines the index file formats used in this version of Lucene. If you are using a different version of Lucene, please consult the copy of docs/ that was distributed with the version you are using.
> This document attempts to provide a high-level definition of the Apache Lucene file formats.

主要介绍索引文件的格式定义
[Apache原文链接](https://lucene.apache.org/core/8_7_0/core/org/apache/lucene/codecs/lucene87/package-summary.html#package.description)


## Definitions
> The fundamental concepts in Lucene are index, document, field and term.

索引、文档、域、词项。
> An index contains a sequence of documents.
> 一个索引包含一个文档序列

> A document is a sequence of fields.
> 一个文档包含一系列域

> A field is a named sequence of terms.
> 一个域包含一系列词项

> A term is a sequence of bytes.
> 一个term是一个字节序列，Token是一种在对文本进行分词时产生的对象，包含分词对象（Term）的词语内容，词语在文本中的开始、结束位置，和一个词语类型（关键字、停用词）字符串。

The same sequence of bytes in two different fields is considered a different term. Thus terms are represented as a pair: the string naming the field, and the bytes within the field.

## Inverted Indexing
> The index stores statistics about terms in order to make term-based search more efficient. Lucene's index falls into the family of indexes known as an inverted index. This is because it can list, for a term, the documents that contain it. This is the inverse of the natural relationship, in which documents list terms.

倒排索引存储了terms的统计，使基于term的搜索更加高效。Lucene的索引属于倒排索引

### Types of Fields

> In  Lucene, fields may be stored, in which case their text is stored in the index literally, in a non-inverted manner. Fields that are inverted are called indexed. A field may be both stored and indexed.

Lucene 中 域可以有序的被存储在索引中，倒排后的域被称为已索引，一个域可以同时被存储和索引。

> The text of a field may be tokenized into terms to be indexed, or the text of a field may be used literally as a term to be indexed. Most fields are tokenized, but sometimes it is useful for certain identifier fields to be indexed literally.

text域可以被标记成要索引的terms，或者一个域的text可以有序的作为一个索引的term。

### Segments

> Lucene indexes may be composed of multiple sub-indexes, or segments. Each segment is a fully independent index, which could be searched separately. Indexes evolve by:
>
> Creating new segments for newly added documents.
> Merging existing segments.
> Searches may involve multiple segments and/or multiple indexes, each index potentially composed of a set of segments.

Lucene 索引可以组成多个子索引或者段。每一个段是一个完全独立的索引，可以单独被搜索到。

索引通过以下方式演变：

1. 为新添加的文档创建新段。
2. 合并现有段。

搜索可能涉及多个段和/或多个索引，每个索引可能由一组段组成。

### Document Numbers-文档编号

> Internally, Lucene refers to documents by an integer document number. The first document added to an index is numbered zero, and each subsequent document added gets a number one greater than the previous.

Lucene内部，文档通过整型文档编号与文档关联。第一个文档入到索引中文档编号为0，每一个后续索引的文档会在之前的文档编号基础上加1

> Note that a document's number may change, so caution should be taken when storing these numbers outside of Lucene. In particular, numbers may change in the following situations:
>
> 一个文档的编号可能会改变，所以从外部存储到Lucene的时候要格外注意。下面的几种情况可能会改变文档编号：

- The numbers stored in each segment are unique only within the segment, and must be converted before they can be used in a larger context. The standard technique is to allocate each segment a range of values, based on the range of numbers used in that segment. To convert a document number from a segment to an external value, the segment's base document number is added. To convert an external value back to a segment-specific value, the segment is identified by the range that the external value is in, and the segment's base value is subtracted. For example two five document segments might be combined, so that the first segment has a base value of zero, and the second of five. Document three from the second segment would have an external value of eight.

  只有存储在每个段内的编号是唯一的，必须先进行转换，才能在更大的上下文中使用。标准的方法是给每个段分配一个值的范围，要基于这个段去分配。要将文档编号转换为外部值，需添加段的*基本*单据编号。要将外部值转换回段特定值，段由外部值所在的范围标识，并减去段的基值。例如，将两个5篇文档的段合并，第一个段的基础值为0，第二个短的基础值为5。第二个段中的第三篇文档的有个外部值为8。

- When documents are deleted, gaps are created in the numbering. These are eventually removed as the index evolves through merging. Deleted documents are dropped when segments are merged. A freshly-merged segment thus has no gaps in its numbering.

  文档被删除时，会产生序列上的间隙。这些间隙最终会在段合并的过程中通过merge处理掉。

### Index Structure Overview ---所以结构概述

Each segment index maintains the following:

**Segment info**. This contains metadata about a segment, such as the number of documents, what files it uses, and information about how the segment is sorted

段信息：包含段元数据信息，例如：文档数，使用的文件，段排序信息

**Field names**. This contains the set of field names used in the index.

域名称：包含当前索引中所有的域名称

**Stored Field values**. This contains, for each document, a list of attribute-value pairs, where the attributes are field names. These are used to store auxiliary information about the document, such as its title, url, or an identifier to access a database. The set of stored fields are what is returned for each hit when searching. This is keyed by document number.

存储的域值：包含每篇文档，属性-值 对的列表：属性为域名，用来存储文档的辅助信息，比如title，url，或者访问数据库的identifier。这个存储字段集合包含搜索时的hit。这个和文档编号关联的。

**Term dictionary**. A dictionary containing all of the terms used in all of the indexed fields of all of the documents. The dictionary also contains the number of documents which contain the term, and pointers to the term's frequency and proximity data.

term词典：包含所有文档的所有被索引域的所有term。该词典还包含包含该术语的文档数量，以及指向该术语的频率和接近度数据的指针。

**Term Frequency data**. For each term in the dictionary, the numbers of all the documents that contain that term, and the frequency of the term in that document, unless frequencies are omitted (IndexOptions.DOCS)

对于字典中的每个术语，包含该术语的所有文档的编号，以及该术语在该文档中的频率，除非频率被省略

**Term Proximity data**. For each term in the dictionary, the positions that the term occurs in each document. Note that this will not exist if all fields in all documents omit position data.
**Normalization factors**. For each field in each document, a value is stored that is multiplied into the score for hits on that field.
**Term Vectors**. For each field in each document, the term vector (sometimes called document vector) may be stored. A term vector consists of term text and term frequency. To add Term Vectors to your index see the Field constructors
**Per-document values**. Like stored values, these are also keyed by document number, but are generally intended to be loaded into main memory for fast access. Whereas stored values are generally intended for summary results from searches, per-document values are useful for things like scoring factors.
**Live documents**. An optional file indicating which documents are live.
**Point values**. Optional pair of files, recording dimensionally indexed fields, to enable fast numeric range filtering and large numeric values like BigInteger and BigDecimal (1D) and geographic shape intersection (2D, 3D).
Details on each of these are provided in their linked pages.

记录维度索引字段，以启用快速数值范围过滤和大数值像BigInteger、BigDecimal以及地理位置（2D,3D）

### **File Naming**

All files belonging to a segment have the same name with varying extensions. The extensions correspond to the different file formats described below. When using the Compound File format (default for small segments) these files (except for the Segment info file, the Lock file, and Deleted documents file) are collapsed into a single .cfs file (see below for details)

属于一个段的所有文件都具有相同的名称和不同的扩展名。扩展名对应于下面描述的不同文件格式。使用复合文件格式（默认用于小段）时，这些文件（段信息文件、锁定文件和已删除文档文件除外）将折叠成单个 .cfs 文件（详情见下文

Typically, all segments in an index are stored in a single directory, although this is not required.

File names are never re-used. That is, when any file is saved to the Directory it is given a never before used filename. This is achieved using a simple generations approach. For example, the first segments file is segments_1, then segments_2, etc. The generation is a sequential long integer represented in alpha-numeric (base 36) form.

通常，索引中的所有段都存储在单个目录中，尽管这不是必需的。

永远不会重复使用文件名。也就是说，当任何文件被保存到目录时，它会被赋予一个以前从未使用过的文件名。这是使用简单的生成方法实现的。例如，第一个段文件是segments_1，然后是segments_2，等等。生成是一个连续的长整数，以字母数字（基数为36）的形式表示。

### Summary of File Extensions--文件扩展名摘要

The following table summarizes the names and extensions of the files in Lucene:

lucene filenames by extension

#### Name	Extension	Brief Description

**Segments File**	**segments_N**	Stores information about a commit point

段文件	segments_N	存储有关提交点的信息

**Lock File**	write.lock	The Write lock prevents multiple IndexWriters from writing to the same file.

锁文件：写锁防止多个IndexWriter写入同一个文件

**Segment Info**	.si	Stores metadata about a segment

段信息：存储段的元数据信息

**Compound File**	.cfs, .cfe	An optional "virtual" file consisting of all the other index files for systems that frequently run out of file handles.

复合文件：一个可选的”虚拟“文件，控制文件句柄过多。

**Fields**	.fnm	Stores information about the fields

域：存储域信息

**Field Index**	.fdx	Contains pointers to field data

域索引：包含域值的指针

**Field Data**	.fdt	The stored fields for documents

域数据：文档存储的所有的域

**Term Dictionary**	.tim	The term dictionary, stores term info

词项词典：term词典，存储term信息

Term Index	.tip	The index into the Term Dictionary

term索引：term词典中的索引

Frequencies	.doc	Contains the list of docs which contain each term along with frequency

频数：包含文档列表中每个term和词频

Positions	.pos	Stores position information about where a term occurs in the index

位置信息：存储term在索引中的位置信息。

Payloads	.pay	Stores additional per-position metadata information such as character offsets and user payloads

存储额外的每个位置元数据信息，例如字符偏移量和用户的payloads

Norms	.nvd, .nvm	Encodes length and boost factors for docs and fields

Norms：编码文档和域的长度和提升因子

Per-Document Values	.dvd, .dvm	Encodes additional scoring factors or other per-document information.

编码额外的评分因子或其他每个文档的信息

Term Vector Index	.tvx	Stores offset into the document data file

存储偏移量到文档数据文件

Term Vector Data	.tvd	Contains term vector data.

包含term向量文件

Live Documents	.liv	Info about what documents are live

有关哪些文档处于活动状态的信息

Point values	.dii, .dim	Holds indexed points, if any

存储索引的points

### Lock File

The write lock, which is stored in the index directory by default, is named "write.lock". If the lock directory is different from the index directory then the write lock will be named "XXXX-write.lock" where XXXX is a unique prefix derived from the full path to the index directory. When this file is present, a writer is currently modifying the index (adding or removing documents). This lock file ensures that only one writer is modifying the index at a time.

写锁，存储在索引目录的写锁文件，默认命名为write.lock。如果这个锁目录和索引目录不同，写锁会命名为”XXX-write.lock“，其中XXX是索引目录的完整路径派生的唯一前缀。当此文件存在时，writer当前正在修改索引（增加或者删除文档），这个锁文件确保只有一个writer同时修改一个索引。