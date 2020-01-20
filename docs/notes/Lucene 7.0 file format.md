```java
package org.apache.lucene.codecs.lucene70;
```

Lucene 7.0 file format.

# Apache Lucene - Index File Formats

[Introduction](#Introduction)
Definitions
Inverted Indexing
[Types of Fields](#Types of Fields)
Segments
Document Numbers
Index Structure Overview
File Naming
Summary of File Extensions
Lock File
History
Limitations

## <a id="Introduction">Introduction</a>

This document defines the index file formats used in this version of Lucene. If you are using a different version of Lucene, please consult the copy of docs/ that was distributed with the version you are using.
This document attempts to provide a high-level definition of the Apache Lucene file formats.

## Definitions

The fundamental concepts in Lucene are index, document, field and term.
An index contains a sequence of documents.

- A document is a sequence of fields.
- A field is a named sequence of terms.
- A term is a sequence of bytes.

The same sequence of bytes in two different fields is considered a different term. Thus terms are represented as a pair: the string naming the field, and the bytes within the field.

### Inverted Indexing

The index stores statistics about terms in order to make term-based search more efficient. Lucene's index falls into the family of indexes known as an inverted index. This is because it can list, for a term, the documents that contain it. This is the inverse of the natural relationship, in which documents list terms.

### <a id="Types of Fields">Types of Fields</a>

In Lucene, fields may be stored, in which case their text is stored in the index literally, in a non-inverted manner. Fields that are inverted are called indexed. A field may be both stored and indexed.
The text of a field may be tokenized into terms to be indexed, or the text of a field may be used literally as a term to be indexed. Most fields are tokenized, but sometimes it is useful for certain identifier fields to be indexed literally.
See the Field java docs for more information on Fields.

### Segments

Lucene indexes may be composed of multiple sub-indexes, or segments. Each segment is a fully independent index, which could be searched separately. Indexes evolve by:

1. Creating new segments for newly added documents.
2. Merging existing segments.

Searches may involve multiple segments and/or multiple indexes, each index potentially composed of a set of segments.

### Document Numbers

Internally, Lucene refers to documents by an integer document number. The first document added to an index is numbered zero, and each subsequent document added gets a number one greater than the previous.
Note that a document's number may change, so caution should be taken when storing these numbers outside of Lucene. In particular, numbers may change in the following situations:

- The numbers stored in each segment are unique only within the segment, and must be converted before they can be used in a larger context. The standard technique is to allocate each segment a range of values, based on the range of numbers used in that segment. To convert a document number from a segment to an external value, the segment's base document number is added. To convert an external value back to a segment-specific value, the segment is identified by the range that the external value is in, and the segment's base value is subtracted. For example two five document segments might be combined, so that the first segment has a base value of zero, and the second of five. Document three from the second segment would have an external value of eight.
- When documents are deleted, gaps are created in the numbering. These are eventually removed as the index evolves through merging. Deleted documents are dropped when segments are merged. A freshly-merged segment thus has no gaps in its numbering.

## Index Structure Overview

Each segment index maintains the following:

[Segment info](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene62/Lucene62SegmentInfoFormat.html). This contains metadata about a segment, such as the number of documents, what files it uses,
[Field names](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50FieldInfosFormat.html). This contains the set of field names used in the index.
[Stored Field values](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html). This contains, for each document, a list of attribute-value pairs, where the attributes are field names. These are used to store auxiliary information about the document, such as its title, url, or an identifier to access a database. The set of stored fields are what is returned for each hit when searching. This is keyed by document number.
[Term dictionary](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html). A dictionary containing all of the terms used in all of the indexed fields of all of the documents. The dictionary also contains the number of documents which contain the term, and pointers to the term's frequency and proximity data.
[Term Frequency data](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html). For each term in the dictionary, the numbers of all the documents that contain that term, and the frequency of the term in that document, unless frequencies are omitted (IndexOptions.DOCS_ONLY)
[Term Proximity data](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html). For each term in the dictionary, the positions that the term occurs in each document. Note that this will not exist if all fields in all documents omit position data.
[Normalization factors](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene70/Lucene70NormsFormat.html). For each field in each document, a value is stored that is multiplied into the score for hits on that field.
[Term Vectors](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50TermVectorsFormat.html). For each field in each document, the term vector (sometimes called document vector) may be stored. A term vector consists of term text and term frequency. To add Term Vectors to your index see the Field constructors
[Per-document values](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene70/Lucene70DocValuesFormat.html). Like stored values, these are also keyed by document number, but are generally intended to be loaded into main memory for fast access. Whereas stored values are generally intended for summary results from searches, per-document values are useful for things like scoring factors.
[Live documents](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50LiveDocsFormat.html). An optional file indicating which documents are live.
[Point values](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene60/Lucene60PointsFormat.html). Optional pair of files, recording dimensionally indexed fields, to enable fast numeric range filtering and large numeric values like BigInteger and BigDecimal (1D) and geographic shape intersection (2D, 3D).

Details on each of these are provided in their linked pages.

## File Naming

All files belonging to a segment have the same name with varying extensions. The extensions correspond to the different file formats described below. When using the Compound File format (default for small segments) these files (except for the Segment info file, the Lock file, and Deleted documents file) are collapsed into a single .cfs file (see below for details)
Typically, all segments in an index are stored in a single directory, although this is not required.
File names are never re-used. That is, when any file is saved to the Directory it is given a never before used filename. This is achieved using a simple generations approach. For example, the first segments file is segments_1, then segments_2, etc. The generation is a sequential long integer represented in alpha-numeric (base 36) form.

## Summary of File Extensions

The following table summarizes the names and extensions of the files in Lucene:

| Name                                                         | Extension                                                    | Brief Description                                            |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Segments File](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/SegmentInfos.html) | [segments_N](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0610/65.html) | Stores information about a commit point                      |
| Lock File                                                    | [write.lock](https://www.amazingkoala.com.cn/Lucene/Store/2019/0604/62.html) | The Write lock prevents multiple IndexWriters from writing to the same file. |
| [Segment Info](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene62/Lucene62SegmentInfoFormat.html) | [.si](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0605/63.html) | Stores metadata about a segment                              |
| [Compound File](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50CompoundFormat.html) | [.cfs, .cfe](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0710/73.html) | An optional "virtual" file consisting of all the other index files for systems that frequently run out of file handles. |
| [Fields](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50FieldInfosFormat.html) | [.fnm](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0606/64.html) | Stores information about the fields                          |
| [Field Index](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html) | [.fdx](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0301/38.html) | Contains pointers to field data                              |
| [Field Data](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html) | [.fdt](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0301/38.html) | The stored fields for documents                              |
| [Term Dictionary](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | [.tim](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0401/43.html) | The term dictionary, stores term info                        |
| [Term Index](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | [.tip](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0401/43.html) | The index into the Term Dictionary                           |
| [Frequencies](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | [.doc](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0324/42.html) | Contains the list of docs which contain each term along with frequency |
| [Positions](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | [.pos](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0324/41.html) | Stores position information about where a term occurs in the index |
| [Payloads](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | [.pay](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0324/41.html) | Stores additional per-position metadata information such as character offsets and user payloads |
| [Norms](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene70/Lucene70NormsFormat.html) | [.nvd, .nvm](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0305/39.html) | Encodes length and boost factors for docs and fields         |
| [Per-Document Values](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene70/Lucene70DocValuesFormat.html) | [.dvd, .dvm](https://www.amazingkoala.com.cn/Lucene/DocValues/2019/0409/46.html) | Encodes additional scoring factors or other per-document information. |
| [Term Vector Index](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50TermVectorsFormat.html) | [.tvx](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0429/56.html) | Stores offset into the document data file                    |
| [Term Vector Data](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50TermVectorsFormat.html) | [.tvd](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0429/56.html) | Contains term vector data.                                   |
| [Live Documents](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene50/Lucene50LiveDocsFormat.html) | [.liv](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0425/54.html) | Info about what documents are live                           |
| [Point values](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/lucene60/Lucene60PointsFormat.html) | [.dii, .dim](https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0424/53.html) | Holds indexed points, if any                                 |

![](pics\Lucen70.png)

### Lock File

The write lock, which is stored in the index directory by default, is named "write.lock". If the lock directory is different from the index directory then the write lock will be named "XXXX-write.lock" where XXXX is a unique prefix derived from the full path to the index directory. When this file is present, a writer is currently modifying the index (adding or removing documents). This lock file ensures that only one writer is modifying the index at a time. 

### History

Compatibility notes are provided in this document, describing how file formats have changed from prior versions:

**In version 2.1**, the file format was changed to allow lock-less commits (ie, no more commit lock). The change is fully backwards compatible: you can open a pre-2.1 index for searching or adding/deleting of docs. When the new segments file is saved (committed), it will be written in the new file format (meaning no specific "upgrade" process is needed). But note that once a commit has occurred, pre-2.1 Lucene will not be able to read the index.
**In version 2.3**, the file format was changed to allow segments to share a single set of doc store (vectors & stored fields) files. This allows for faster indexing in certain cases. The change is fully backwards compatible (in the same way as the lock-less commits change in 2.1).
**In version 2.4**, Strings are now written as true UTF-8 byte sequence, not Java's modified UTF-8. See [LUCENE-510](https://issues.apache.org/jira/browse/LUCENE-510)  for details.
**In version 2.9**, an optional opaque Map<String,String> CommitUserData may be passed to IndexWriter's commit methods (and later retrieved), which is recorded in the segments_N file. See [LUCENE-138](https://issues.apache.org/jira/browse/LUCENE-1382)2  for details. Also, diagnostics were added to each segment written recording details about why it was written (due to flush, merge; which OS/JRE was used; etc.). See issue [LUCENE-1654](https://issues.apache.org/jira/browse/LUCENE-1654)  for details.
**In version 3.0**, compressed fields are no longer written to the index (they can still be read, but on merge the new segment will write them, uncompressed). See issue [LUCENE-1960](https://issues.apache.org/jira/browse/LUCENE-1960)  for details.
**In version 3.1**, segments records the code version that created them. See [LUCENE-2720](https://issues.apache.org/jira/browse/LUCENE-2720)  for details. Additionally segments track explicitly whether or not they have term vectors. See [LUCENE-2811](https://issues.apache.org/jira/browse/LUCENE-2811)  for details.
**In version 3.2**, numeric fields are written as natively to stored fields file, previously they were stored in text format only.
**In version 3.4**, fields can omit position data while still indexing term frequencies.
**In version 4.0**, the format of the inverted index became extensible via the [Codec](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/Codec.html) api. Fast per-document storage (DocValues) was introduced. Normalization factors need no longer be a single byte, they can be any [NumericDocValues](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/NumericDocValues.html). Terms need not be unicode strings, they can be any byte sequence. Term offsets can optionally be indexed into the postings lists. Payloads can be stored in the term vectors.
**In version 4.1**, the format of the postings list changed to use either of FOR compression or variable-byte encoding, depending upon the frequency of the term. Terms appearing only once were changed to inline directly into the term dictionary. Stored fields are compressed by default.
**In version 4.2**, term vectors are compressed by default. DocValues has a new multi-valued type (SortedSet), that can be used for faceting/grouping/joining on multi-valued fields.
**In version 4.5**, DocValues were extended to explicitly represent missing values.
**In version 4.6**, FieldInfos were extended to support per-field DocValues generation, to allow updating NumericDocValues fields.
**In version 4.8**, checksum footers were added to the end of each index file for improved data integrity. Specifically, the last 8 bytes of every index file contain the zlib-crc32 checksum of the file.
**In version 4.9**, DocValues has a new multi-valued numeric type (SortedNumeric) that is suitable for faceting/sorting/analytics.
**In version 5.4**, DocValues have been improved to store more information on disk: addresses for binary fields and ord indexes for multi-valued fields.
**In version 6.0**, Points were added, for multi-dimensional range/distance search.
**In version 6.2**, new Segment info format that reads/writes the index sort, to support index sorting.
**In version 7.0**, DocValues have been improved to better support sparse doc values thanks to an iterator API.

### Limitations

Lucene uses a Java int to refer to document numbers, and the index file format uses an Int32 on-disk to store document numbers. This is a limitation of both the index file format and the current implementation. Eventually these should be replaced with either UInt64 values, or better yet, VInt values which have no limit.