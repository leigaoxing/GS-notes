## Package org.apache.lucene.index Description

Code to maintain and access indices.

## Table Of Contents

1. Index APIs
   - [IndexWriter](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#writer)
   - [IndexReader](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#reader)
   - [Segments and docids](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#segments)
2. Field types
   - [Postings](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#postings-desc)
   - [Stored Fields](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#stored-fields)
   - [DocValues](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#docvalues)
   - [Points](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#points)
3. Postings APIs
   - [Fields](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#fields)
   - [Terms](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#terms)
   - [Documents](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#documents)
   - [Positions](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#positions)
4. Index Statistics
   - [Term-level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#termstats)
   - [Field-level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#fieldstats)
   - [Segment-level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#segmentstats)
   - [Document-level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#documentstats)



## Index APIs



### IndexWriter

[`IndexWriter`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexWriter.html) is used to create an index, and to add, update and delete documents. The IndexWriter class is thread safe, and enforces a single instance per index. Creating an IndexWriter creates a new index or opens an existing index for writing, in a [`Directory`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/Directory.html), depending on the configuration in [`IndexWriterConfig`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexWriterConfig.html). A Directory is an abstraction that typically represents a local file-system directory (see various implementations of [`FSDirectory`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/FSDirectory.html)), but it may also stand for some other storage, such as RAM.



### IndexReader

[`IndexReader`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexReader.html) is used to read data from the index, and supports searching. Many thread-safe readers may be [`DirectoryReader.open(org.apache.lucene.store.Directory)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/DirectoryReader.html#open-org.apache.lucene.store.Directory-) concurrently with a single (or no) writer. Each reader maintains a consistent "point in time" view of an index and must be explicitly refreshed (see [`DirectoryReader.openIfChanged(org.apache.lucene.index.DirectoryReader)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/DirectoryReader.html#openIfChanged-org.apache.lucene.index.DirectoryReader-)) in order to incorporate writes that may occur after it is opened.



### Segments and docids

Lucene's index is composed of segments, each of which contains a subset of all the documents in the index, and is a complete searchable index in itself, over that subset. As documents are written to the index, new segments are created and flushed to directory storage. Segments are immutable; updates and deletions may only create new segments and do not modify existing ones. Over time, the writer merges groups of smaller segments into single larger ones in order to maintain an index that is efficient to search, and to reclaim dead space left behind by deleted (and updated) documents.

Each document is identified by a 32-bit number, its "docid," and is composed of a collection of Field values of diverse types (postings, stored fields, doc values, and points). Docids come in two flavors: global and per-segment. A document's global docid is just the sum of its per-segment docid and that segment's base docid offset. External, high-level APIs only handle global docids, but internal APIs that reference a [`LeafReader`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/LeafReader.html), which is a reader for a single segment, deal in per-segment docids.

Docids are assigned sequentially within each segment (starting at 0). Thus the number of documents in a segment is the same as its maximum docid; some may be deleted, but their docids are retained until the segment is merged. When segments merge, their documents are assigned new sequential docids. Accordingly, docid values must always be treated as internal implementation, not exposed as part of an application, nor stored or referenced outside of Lucene's internal APIs.



## Field Types



Lucene supports a variety of different document field data structures. Lucene's core, the inverted index, is comprised of "postings." The postings, with their term dictionary, can be thought of as a map that provides efficient lookup given a [`Term`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Term.html) (roughly, a word or token), to (the ordered list of) [`Document`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Document.html)s containing that Term. Postings do not provide any way of retrieving terms given a document, short of scanning the entire index.



Stored fields are essentially the opposite of postings, providing efficient retrieval of field values given a docid. All stored field values for a document are stored together in a block. Different types of stored field provide high-level datatypes such as strings and numbers on top of the underlying bytes. Stored field values are usually retrieved by the searcher using an implementation of [`StoredFieldVisitor`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/StoredFieldVisitor.html).



[`DocValues`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/DocValues.html) fields are what are sometimes referred to as columnar, or column-stride fields, by analogy to relational database terminology, in which documents are considered as rows, and fields, columns. DocValues fields store values per-field: a value for every document is held in a single data structure, providing for rapid, sequential lookup of a field-value given a docid. These fields are used for efficient value-based sorting, and for faceting, but they are not useful for filtering.



[`PointValues`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/PointValues.html) represent numeric values using a kd-tree data structure. Efficient 1- and higher dimensional implementations make these the choice for numeric range and interval queries, and geo-spatial queries.



## Postings APIs



### Fields

[`Fields`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Fields.html) is the initial entry point into the postings APIs, this can be obtained in several ways:

```java
 // access indexed fields for an index segment
 Fields fields = reader.fields();
 // access term vector fields for a specified document
 Fields fields = reader.getTermVectors(docid);
 
```

Fields implements Java's Iterable interface, so it's easy to enumerate the list of fields:

```java
 // enumerate list of fields
 for (String field : fields) {
   // access the terms for this field
   Terms terms = fields.terms(field);
 }
 
```



### Terms

[`Terms`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Terms.html) represents the collection of terms within a field, exposes some metadata and [statistics](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/package-summary.html#fieldstats), and an API for enumeration.

```java
 // metadata about the field
 System.out.println("positions? " + terms.hasPositions());
 System.out.println("offsets? " + terms.hasOffsets());
 System.out.println("payloads? " + terms.hasPayloads());
 // iterate through terms
 TermsEnum termsEnum = terms.iterator(null);
 BytesRef term = null;
 while ((term = termsEnum.next()) != null) {
   doSomethingWith(termsEnum.term());
 }
 
```

[`TermsEnum`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/TermsEnum.html)      .

[`PostingsEnum`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/PostingsEnum.html) is an extension of [`DocIdSetIterator`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/DocIdSetIterator.html)that iterates over the list of documents for a term, along with the term frequency within that document.

```java
 int docid;
 while ((docid = docsEnum.nextDoc()) != DocIdSetIterator.NO_MORE_DOCS) {
   System.out.println(docid);
   System.out.println(docsEnum.freq());
  }
 
```



### Positions

PostingsEnum also allows iteration of the positions a term occurred within the document, and any additional per-position information (offsets and payload). The information available is controlled by flags passed to TermsEnum#postings

```java
 int docid;
 PostingsEnum postings = termsEnum.postings(null, null, PostingsEnum.FLAG_PAYLOADS | PostingsEnum.FLAG_OFFSETS);
 while ((docid = postings.nextDoc()) != DocIdSetIterator.NO_MORE_DOCS) {
   System.out.println(docid);
   int freq = postings.freq();
   for (int i = 0; i < freq; i++) {
    System.out.println(postings.nextPosition());
    System.out.println(postings.startOffset());
    System.out.println(postings.endOffset());
    System.out.println(postings.getPayload());
   }
 }
 
```



## Index Statistics



### Term statistics

- [`TermsEnum.docFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/TermsEnum.html#docFreq--): Returns the number of documents that contain at least one occurrence of the term. This statistic is always available for an indexed term. Note that it will also count deleted documents, when segments are merged the statistic is updated as those deleted documents are merged away.
- [`TermsEnum.totalTermFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/TermsEnum.html#totalTermFreq--): Returns the number of occurrences of this term across all documents. Note that this statistic is unavailable (returns `-1`) if term frequencies were omitted from the index ([`DOCS`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexOptions.html#DOCS)) for the field. Like docFreq(), it will also count occurrences that appear in deleted documents.



### Field statistics

- [`Terms.size()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Terms.html#size--): Returns the number of unique terms in the field. This statistic may be unavailable (returns `-1`) for some Terms implementations such as [`MultiTerms`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/MultiTerms.html), where it cannot be efficiently computed. Note that this count also includes terms that appear only in deleted documents: when segments are merged such terms are also merged away and the statistic is then updated.
- [`Terms.getDocCount()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Terms.html#getDocCount--): Returns the number of documents that contain at least one occurrence of any term for this field. This can be thought of as a Field-level docFreq(). Like docFreq() it will also count deleted documents.
- [`Terms.getSumDocFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Terms.html#getSumDocFreq--): Returns the number of postings (term-document mappings in the inverted index) for the field. This can be thought of as the sum of [`TermsEnum.docFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/TermsEnum.html#docFreq--) across all terms in the field, and like docFreq() it will also count postings that appear in deleted documents.
- [`Terms.getSumTotalTermFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Terms.html#getSumTotalTermFreq--): Returns the number of tokens for the field. This can be thought of as the sum of [`TermsEnum.totalTermFreq()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/TermsEnum.html#totalTermFreq--) across all terms in the field, and like totalTermFreq() it will also count occurrences that appear in deleted documents, and will be unavailable (returns `-1`) if term frequencies were omitted from the index ([`DOCS`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexOptions.html#DOCS)) for the field.



### Segment statistics

- [`IndexReader.maxDoc()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexReader.html#maxDoc--): Returns the number of documents (including deleted documents) in the index.
- [`IndexReader.numDocs()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexReader.html#numDocs--): Returns the number of live documents (excluding deleted documents) in the index.
- [`IndexReader.numDeletedDocs()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexReader.html#numDeletedDocs--): Returns the number of deleted documents in the index.
- [`Fields.size()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Fields.html#size--): Returns the number of indexed fields.



### Document statistics

Document statistics are available during the indexing process for an indexed field: typically a [`Similarity`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html) implementation will store some of these values (possibly in a lossy way), into the normalization value for the document in its [`Similarity.computeNorm(org.apache.lucene.index.FieldInvertState)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html#computeNorm-org.apache.lucene.index.FieldInvertState-) method.

- [`FieldInvertState.getLength()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getLength--): Returns the number of tokens for this field in the document. Note that this is just the number of times that [`TokenStream.incrementToken()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/TokenStream.html#incrementToken--) returned true, and is unrelated to the values in[`PositionIncrementAttribute`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/tokenattributes/PositionIncrementAttribute.html).
- [`FieldInvertState.getNumOverlap()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getNumOverlap--): Returns the number of tokens for this field in the document that had a position increment of zero. This can be used to compute a document length that discounts artificial tokens such as synonyms.
- [`FieldInvertState.getPosition()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getPosition--): Returns the accumulated position value for this field in the document: computed from the values of [`PositionIncrementAttribute`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/tokenattributes/PositionIncrementAttribute.html) and including[`Analyzer.getPositionIncrementGap(java.lang.String)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/Analyzer.html#getPositionIncrementGap-java.lang.String-)s across multivalued fields.
- [`FieldInvertState.getOffset()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getOffset--): Returns the total character offset value for this field in the document: computed from the values of [`OffsetAttribute`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/tokenattributes/OffsetAttribute.html) returned by [`TokenStream.end()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/TokenStream.html#end--), and including[`Analyzer.getOffsetGap(java.lang.String)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/analysis/Analyzer.html#getOffsetGap-java.lang.String-)s across multivalued fields.
- [`FieldInvertState.getUniqueTermCount()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getUniqueTermCount--): Returns the number of unique terms encountered for this field in the document.
- [`FieldInvertState.getMaxTermFrequency()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/FieldInvertState.html#getMaxTermFrequency--): Returns the maximum frequency across all unique terms encountered for this field in the document.

Additional user-supplied statistics can be added to the document as DocValues fields and accessed via [`LeafReader.getNumericDocValues(java.lang.String)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/LeafReader.html#getNumericDocValues-java.lang.String-).