## Package org.apache.lucene.search Description

Code to search indices.

## Table Of Contents

1. [Search Basics](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#search)
2. [The Query Classes](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#query)
3. [Scoring: Introduction](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#scoring)
4. [Scoring: Basics](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#scoringBasics)
5. [Changing the Scoring](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#changingScoring)
6. [Appendix: Search Algorithm](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#algorithm)



Lucene offers a wide variety of [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) implementations, most of which are in this package, its subpackage ([`spans`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/spans/package-summary.html), or the [queries module](http://lucene.apache.org/core/7_5_0/queries/overview-summary.html). These implementations can be combined in a wide variety of ways to provide complex querying capabilities along with information about where matches took place in the document collection. The [Query Classes](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#query) section below highlights some of the more important Query classes. For details on implementing your own Query class, see [Custom Queries -- Expert Level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#customQueriesExpert) below.

To perform a search, applications usually call [`IndexSearcher.search(Query,int)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html#search-org.apache.lucene.search.Query-int-).

Once a Query has been created and submitted to the [`IndexSearcher`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html), the scoring process begins. After some infrastructure setup, control finally passes to the [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) implementation and its [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) or [`BulkScore`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html) instances. See the [Algorithm](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#algorithm)section for more notes on the process.

## Query Classes

### [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html)

Of the various implementations of [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html), the [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) is the easiest to understand and the most often used in applications. A [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) matches all the documents that contain the specified [`Term`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Term.html), which is a word that occurs in a certain[`Field`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Field.html). Thus, a [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) identifies and scores all [`Document`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Document.html)s that have a [`Field`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Field.html) with the specified string in it. Constructing a [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) is as simple as:

```
         TermQuery tq = new TermQuery(new Term("fieldName", "term"));
     
```

In this example, the [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html)[`Document`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Document.html)[`Field`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Field.html)`"fieldName"``"term"`

### [`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html)

Things start to get interesting when one combines multiple [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) instances into a [`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html). A [`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html) contains multiple [`BooleanClause`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.html)s, where each clause contains a sub-query ([`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) instance) and an operator (from[`BooleanClause.Occur`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.Occur.html)) describing how that sub-query is combined with the other clauses:

1. [`SHOULD`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.Occur.html#SHOULD) — Use this operator when a clause can occur in the result set, but is not required. If a query is made up of all SHOULD clauses, then every document in the result set matches at least one of these clauses.
2. [`MUST`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.Occur.html#MUST) — Use this operator when a clause is required to occur in the result set. Every document in the result set will match all such clauses.
3. [`MUST NOT`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.Occur.html#MUST_NOT) — Use this operator when a clause must not occur in the result set. No document in the result set will match any such clauses.

Boolean queries are constructed by adding two or more [`BooleanClause`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanClause.html)[`TooManyClauses`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.TooManyClauses.html)[`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html)[`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html)[`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html)[`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html)[`BooleanQuery.setMaxClauseCount(int)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html#setMaxClauseCount-int-)

### Phrases

Another common search is to find documents containing certain phrases. This is handled three different ways:

1. [`PhraseQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PhraseQuery.html) — Matches a sequence of [`Term`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Term.html)s. [`PhraseQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PhraseQuery.html) uses a slop factor to determine how many positions may occur between any two terms in the phrase and still be considered a match. The slop is 0 by default, meaning the phrase must match exactly.
2. [`MultiPhraseQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/MultiPhraseQuery.html) — A more general form of PhraseQuery that accepts multiple Terms for a position in the phrase. For example, this can be used to perform phrase queries that also incorporate synonyms.
3. [`SpanNearQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/spans/SpanNearQuery.html) — Matches a sequence of other [`SpanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/spans/SpanQuery.html) instances. [`SpanNearQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/spans/SpanNearQuery.html) allows for much more complicated phrase queries since it is constructed from other [`SpanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/spans/SpanQuery.html) instances, instead of only [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html) instances.

### [`TermRangeQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermRangeQuery.html)

The [`TermRangeQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermRangeQuery.html) matches all documents that occur in the exclusive range of a lower [`Term`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Term.html) and an upper [`Term`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/Term.html) according to [`BytesRef.compareTo()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/util/BytesRef.html#compareTo-org.apache.lucene.util.BytesRef-). It is not intended for numerical ranges; use [`PointRangeQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PointRangeQuery.html) instead. For example, one could find all documents that have terms beginning with the letters `a` through `c`.

### [`PointRangeQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PointRangeQuery.html)

The [`PointRangeQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PointRangeQuery.html) matches all documents that occur in a numeric range. For PointRangeQuery to work, you must index the values using a one of the numeric fields ([`IntPoint`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/IntPoint.html), [`LongPoint`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/LongPoint.html), [`FloatPoint`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/FloatPoint.html), or [`DoublePoint`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/DoublePoint.html)).

### [`PrefixQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PrefixQuery.html), [`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html), [`RegexpQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/RegexpQuery.html)

While the [`PrefixQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PrefixQuery.html) has a different implementation, it is essentially a special case of the [`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html). The [`PrefixQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/PrefixQuery.html) allows an application to identify all documents with terms that begin with a certain string. The [`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html)generalizes this by allowing for the use of `*` (matches 0 or more characters) and `?` (matches exactly one character) wildcards. Note that the [`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html) can be quite slow. Also note that [`WildcardQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/WildcardQuery.html) should not start with `*` and `?`, as these are extremely slow. Some QueryParsers may not allow this by default, but provide a `setAllowLeadingWildcard` method to remove that protection. The [`RegexpQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/RegexpQuery.html) is even more general than WildcardQuery, allowing an application to identify all documents with terms that match a regular expression pattern.

### [`FuzzyQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/FuzzyQuery.html)

A [`FuzzyQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/FuzzyQuery.html) matches documents that contain terms similar to the specified term. Similarity is determined using [Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance). This type of query can be useful when accounting for spelling variations in the collection.

## Scoring — Introduction

Lucene scoring is the heart of why we all love Lucene. It is blazingly fast and it hides almost all of the complexity from the user. In a nutshell, it works. At least, that is, until it doesn't work, or doesn't work as one would expect it to work. Then we are left digging into Lucene internals or asking for help on [java-user@lucene.apache.org](mailto:java-user@lucene.apache.org) to figure out why a document with five of our query terms scores lower than a different document with only one of the query terms.

While this document won't answer your specific scoring issues, it will, hopefully, point you to the places that can help you figure out the *what* and *why* of Lucene scoring.

Lucene scoring supports a number of pluggable information retrieval [models](http://en.wikipedia.org/wiki/Information_retrieval#Model_types), including:

- [Vector Space Model (VSM)](http://en.wikipedia.org/wiki/Vector_Space_Model)
- [Probabilistic Models](http://en.wikipedia.org/wiki/Probabilistic_relevance_model) such as [Okapi BM25](http://en.wikipedia.org/wiki/Probabilistic_relevance_model_(BM25)) and [DFR](http://en.wikipedia.org/wiki/Divergence-from-randomness_model)
- [Language models](http://en.wikipedia.org/wiki/Language_model)

These models can be plugged in via the [`Similarity API`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/package-summary.html)[Lucene Wiki IR references](http://wiki.apache.org/lucene-java/InformationRetrieval)

The rest of this document will cover [Scoring basics](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#scoringBasics) and explain how to change your [`Similarity`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html). Next, it will cover ways you can customize the lucene internals in [Custom Queries -- Expert Level](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#customQueriesExpert), which gives details on implementing your own [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) class and related functionality. Finally, we will finish up with some reference material in the [Appendix](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#algorithm).

## Scoring — Basics

Scoring is very much dependent on the way documents are indexed, so it is important to understand indexing. (see [Lucene overview](http://lucene.apache.org/core/7_5_0/core/overview-summary.html#overview_description) before continuing on with this section) Be sure to use the useful [`IndexSearcher.explain(Query, doc)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html#explain-org.apache.lucene.search.Query-int-) to understand how the score for a certain matching document was computed.

Generally, the Query determines which documents match (a binary decision), while the Similarity determines how to assign scores to the matching documents.

### Fields and Documents

In Lucene, the objects we are scoring are [`Document`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Document.html)s. A Document is a collection of [`Field`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Field.html)s. Each Field has [`semantics`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/FieldType.html) about how it is created and stored ([`tokenized`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/FieldType.html#tokenized--), [`stored`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/FieldType.html#stored--), etc). It is important to note that Lucene scoring works on Fields and then combines the results to return Documents. This is important because two Documents with the exact same content, but one having the content in two Fields and the other in one Field may return different scores for the same query due to length normalization.

### Score Boosting

Lucene allows influencing the score contribution of various parts of the query by wrapping with [`BoostQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BoostQuery.html).



Changing [`Similarity`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html) is an easy way to influence scoring, this is done at index-time with [`IndexWriterConfig.setSimilarity(Similarity)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexWriterConfig.html#setSimilarity-org.apache.lucene.search.similarities.Similarity-) and at query-time with [`IndexSearcher.setSimilarity(Similarity)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html#setSimilarity-org.apache.lucene.search.similarities.Similarity-). Be sure to use the same Similarity at query-time as at index-time (so that norms are encoded/decoded correctly); Lucene makes no effort to verify this.

You can influence scoring by configuring a different built-in Similarity implementation, or by tweaking its parameters, subclassing it to override behavior. Some implementations also offer a modular API which you can extend by plugging in a different component (e.g. term frequency normalizer).

Finally, you can extend the low level [`Similarity`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html) directly to implement a new retrieval model, or to use external scoring factors particular to your application. For example, a custom Similarity can access per-document values via [`NumericDocValues`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/NumericDocValues.html) and integrate them into the score.

See the [`org.apache.lucene.search.similarities`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/package-summary.html) package documentation for information on the built-in available scoring models and extending or changing Similarity.

## Custom Queries — Expert Level

Custom queries are an expert level task, so tread carefully and be prepared to share your code if you want help.

With the warning out of the way, it is possible to change a lot more than just the Similarity when it comes to matching and scoring in Lucene. Lucene's search is a complex mechanism that is grounded by three main classes:

1. [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) — The abstract object representation of the user's information need.
2. [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) — The internal interface representation of the user's Query, so that Query objects may be reused. This is global (across all segments of the index) and generally will require global statistics (such as docFreq for a given term across all segments).
3. [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) — An abstract class containing common functionality for scoring. Provides both scoring and explanation capabilities. This is created per-segment.
4. [`BulkScorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html) — An abstract class that scores a range of documents. A default implementation simply iterates through the hits from [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html), but some queries such as [`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html) have more efficient implementations.

Details on each of these classes, and their children, can be found in the subsections below.

### The Query Class

In some sense, the [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) class is where it all begins. Without a Query, there would be nothing to score. Furthermore, the Query class is the catalyst for the other scoring classes as it is often responsible for creating them or coordinating the functionality between them. The [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) class has several methods that are important for derived classes:

1. [`createWeight(IndexSearcher searcher, boolean needsScores, float boost)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html#createWeight-org.apache.lucene.search.IndexSearcher-boolean-float-) — A [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) is the internal representation of the Query, so each Query implementation must provide an implementation of Weight. See the subsection on [The Weight Interface](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#weightClass) below for details on implementing the Weight interface.
2. [`rewrite(IndexReader reader)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html#rewrite-org.apache.lucene.index.IndexReader-) — Rewrites queries into primitive queries. Primitive queries are: [`TermQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TermQuery.html), [`BooleanQuery`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BooleanQuery.html), and other queries that implement [`createWeight(IndexSearcher searcher,boolean needsScores, float boost)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html#createWeight-org.apache.lucene.search.IndexSearcher-boolean-float-)



The [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) interface provides an internal representation of the Query so that it can be reused. Any [`IndexSearcher`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html) dependent state should be stored in the Weight implementation, not in the Query class. The interface defines five methods that must be implemented:

1. [`getQuery()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html#getQuery--) — Pointer to the Query that this Weight represents.
2. [`scorer()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html#scorer-org.apache.lucene.index.LeafReaderContext-) — Construct a new [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) for this Weight. See [The Scorer Class](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#scorerClass) below for help defining a Scorer. As the name implies, the Scorer is responsible for doing the actual scoring of documents given the Query.
3. [`bulkScorer()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html#bulkScorer-org.apache.lucene.index.LeafReaderContext-) — Construct a new [`BulkScorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html) for this Weight. See [The BulkScorer Class](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#bulkScorerClass) below for help defining a BulkScorer. This is an optional method, and most queries do not implement it.
4. [`explain(LeafReaderContext context, int doc)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html#explain-org.apache.lucene.index.LeafReaderContext-int-) — Provide a means for explaining why a given document was scored the way it was. Typically a weight such as TermWeight that scores via a [`Similarity`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.html) will make use of the Similarity's implementation: [`SimScorer#explain(int doc, Explanation freq)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.SimScorer.html#explain-int-org.apache.lucene.search.Explanation-).



The [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) abstract class provides common scoring functionality for all Scorer implementations and is the heart of the Lucene scoring process. The Scorer defines the following methods which must be implemented:

1. [`iterator()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html#iterator--) — Return a [`DocIdSetIterator`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/DocIdSetIterator.html) that can iterate over all document that matches this Query.
2. [`docID()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html#docID--) — Returns the id of the [`Document`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/document/Document.html) that contains the match.
3. [`score()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html#score--) — Return the score of the current document. This value can be determined in any appropriate way for an application. For instance, the `TermScorer` simply defers to the configured Similarity: [`SimScorer.score(int doc, float freq)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/similarities/Similarity.SimScorer.html#score-int-float-).
4. [`getChildren()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html#getChildren--) — Returns any child subscorers underneath this scorer. This allows for users to navigate the scorer hierarchy and receive more fine-grained details on the scoring process.



The [`BulkScorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html) scores a range of documents. There is only one abstract method:

1. [`score(LeafCollector,Bits,int,int)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html#score-org.apache.lucene.search.LeafCollector-org.apache.lucene.util.Bits-int-int-) — Score all documents up to but not including the specified max document.

### Why would I want to add my own Query?

In a nutshell, you want to add your own custom Query implementation when you think that Lucene's aren't appropriate for the task that you want to do. You might be doing some cutting edge research or you need more information back out of Lucene (similar to Doug adding SpanQuery functionality).

## Appendix: Search Algorithm

This section is mostly notes on stepping through the Scoring process and serves as fertilizer for the earlier sections.

In the typical search application, a [`Query`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Query.html) is passed to the [`IndexSearcher`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html), beginning the scoring process.

Once inside the IndexSearcher, a [`Collector`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Collector.html) is used for the scoring and sorting of the search results. These important objects are involved in a search:

1. The [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) object of the Query. The Weight object is an internal representation of the Query that allows the Query to be reused by the IndexSearcher.
2. The IndexSearcher that initiated the call.
3. A [`Sort`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Sort.html) object for specifying how to sort the results if the standard score-based sort method is not desired.

Assuming we are not sorting (since sorting doesn't affect the raw Lucene score), we call one of the search methods of the IndexSearcher, passing in the [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) object created by [`IndexSearcher.createWeight(Query,ScoreMode,float)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html#createWeight-org.apache.lucene.search.Query-boolean-float-) and the number of results we want. This method returns a [`TopDocs`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TopDocs.html) object, which is an internal collection of search results. The IndexSearcher creates a [`TopScoreDocCollector`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/TopScoreDocCollector.html) and passes it along with the Weight, Filter to another expert search method (for more on the [`Collector`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Collector.html) mechanism, see [`IndexSearcher`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/IndexSearcher.html)). The TopScoreDocCollector uses a [`PriorityQueue`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/util/PriorityQueue.html) to collect the top results for the search.

If a Filter is being used, some initial setup is done to determine which docs to include. Otherwise, we ask the Weight for a [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) for each [`IndexReader`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/index/IndexReader.html) segment and proceed by calling [`BulkScorer.score(LeafCollector,Bits)`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/BulkScorer.html#score-org.apache.lucene.search.LeafCollector-org.apache.lucene.util.Bits-).

At last, we are actually going to score some documents. The score method takes in the Collector (most likely the TopScoreDocCollector or TopFieldCollector) and does its business.Of course, here is where things get involved. The [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html)that is returned by the [`Weight`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Weight.html) object depends on what type of Query was submitted. In most real world applications with multiple query terms, the [`Scorer`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/Scorer.html) is going to be a `BooleanScorer2` created from `BooleanWeight` (see the section on[custom queries](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/package-summary.html#customQueriesExpert) for info on changing this).

Assuming a BooleanScorer2, we get a internal Scorer based on the required, optional and prohibited parts of the query. Using this internal Scorer, the BooleanScorer2 then proceeds into a while loop based on the[`DocIdSetIterator.nextDoc()`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/search/DocIdSetIterator.html#nextDoc--) method. The nextDoc() method advances to the next document matching the query. This is an abstract method in the Scorer class and is thus overridden by all derived implementations. If you have a simple OR query your internal Scorer is most likely a DisjunctionSumScorer, which essentially combines the scorers from the sub scorers of the OR'd terms.