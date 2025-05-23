---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html
applies_to:
  stack:
products:
  - id: elasticsearch
---

# Near real-time search [near-real-time]

When a document is stored in {{es}}, it is indexed and fully searchable in *near real-time*--within 1 second. What defines near real-time search?

Lucene, the Java libraries on which {{es}} is based, introduced the concept of per-segment search. A *segment* is similar to an inverted index, but the word *index* in Lucene means "a collection of segments plus a commit point". After a commit, a new segment is added to the commit point and the buffer is cleared.

Sitting between {{es}} and the disk is the filesystem cache. Documents in the in-memory indexing buffer ([Figure 1](#img-pre-refresh)) are written to a new segment ([Figure 2](#img-post-refresh)). The new segment is written to the filesystem cache first (which is cheap) and only later is it flushed to disk (which is expensive). However, after a file is in the cache, it can be opened and read just like any other file.

:::{image} /manage-data/images/elasticsearch-reference-lucene-in-memory-buffer.png
:alt: A Lucene index with new documents in the in-memory buffer
:title: A Lucene index with new documents in the in-memory buffer
:name: img-pre-refresh
:::

Lucene allows new segments to be written and opened, making the documents they contain visible to search without performing a full commit. This is a much lighter process than a commit to disk, and can be done frequently without degrading performance.

:::{image} /manage-data/images/elasticsearch-reference-lucene-written-not-committed.png
:alt: The buffer contents are written to a segment, which is searchable, but is not yet committed
:title: The buffer contents are written to a segment, which is searchable, but is not yet committed
:name: img-post-refresh
:::

In {{es}}, this process of writing and opening a new segment is called a *refresh*. A refresh makes all operations performed on an index since the last refresh available for search. You can control refreshes through the following means:

* Waiting for the refresh interval
* Setting the [?refresh](elasticsearch://reference/elasticsearch/rest-apis/refresh-parameter.md) option
* Using the [Refresh API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-refresh) to explicitly complete a refresh (`POST _refresh`)

By default, {{es}} periodically refreshes indices every second, but only on indices that have received one search request or more in the last 30 seconds. This is why we say that {{es}} has *near* real-time search: document changes are not visible to search immediately, but will become visible within this timeframe.

% Internal links rely on the following IDs being on this page (e.g. as a heading ID, paragraph ID, etc):

$$$img-pre-refresh$$$

$$$img-post-refresh$$$