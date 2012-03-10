---
layout: post
title: "Evaluating Mark Logic XQuery Performance"
date: 2012-02-12 19:36
comments: true
categories: [XQuery, MarkLogic]
---

I've recently been doing some work building RESTFul API's backed by a Mark Logic XML Content Store utilising XQuery for document retrieval. This post details the steps involved in tuning what was deemed to be the most simplest of queries for optimum performance using some useful Mark Logic extensions for profiling.

### Original Query
--------------
XQuery for looking up documents based on the value of a given attribute in the xml using XPath. (What could be simpler!)
{% gist 1810958 document_by_url.xqy %}

### Evaluating Performance
--------------
By default Mark Logic indexes the document structure and attributes **are** indexed by default as part of the universal index.

#### 1. Adding xdmp:query-meters
By adding xdmp:query-meters() to the query we get some immediate feedback about how the query performs including elapsed time and the number of fragments and documents that were selected. Altering the above query as below gives us some interesting metrics and the query is taking nearly 2 seconds.

{% gist 1810958 document_by_url_with_xdmp_query_meters.xqy %}
{% gist 1810958 result_of_xdmp_query_meters.xml %}

Immediately something looks a bit suspicious as all the documents in the database are being returned which would indicate that the query is not making effective use of Mark Logic's Indexes.
<!-- more -->

#### 2. Verifying with xdmp:estimate
This can be verified with xdmp:estimate(), purely focusing on the XPath part of the query e.g.

{% gist 1810958 document_by_url_with_xdmp_estimate.xqy %}

The evaluator sees the XPath expression above and uses index lookup's to match some sequence of fragments in the database. xdmp:estimate() gives an estimate of the number of documents in a sequence and is directed at the index-lookup phase, i.e "search".

{% gist 1810958 result_of_xdmp_estimate.xml %}

Next, the evaluator will fetch those matching fragment(s), if any, from the database. Now we are back in the evaluation phase. It will check to make sure the nodes really match: this is known as "filtering". Then it will evaluate the entire XPath.

So what we are saying for the xquery above is that the number of matching fragments is all the documents in the database which will then get filtered so we are not making use of any of available Mark Logic indexing which means the query is very inefficient.

#### 3. Looking at the query plan with xdmp:plan
This further verifies that all the documents in the database are being selected and we are not fully leveraging indexes

{% gist 1810958 document_by_url_with_xdmp_plan.xqy %}
{% gist 1810958 result_of_xdmp_plan.xml %}

#### Looking at the XPath in more detail
/* accesses the entire database and returns every root element in the database, but we do it a second time in the predicate which is very expensive.

Changing the XPath to below and re-running the above steps results in a much more positive result, and look how quick the query is!

{% gist 1810958 document_by_url_tuned.xqy %}
{% gist 1810958 result_document_by_url_tuned.xml %}


### Plugging in xinc:node-expand
So far we have done our query evaluation ignoring the final piece which is to plugin the marklogic xinc:node-expand function which will resolves any x:includes in the results
```
return xinc:node-expand($asset) 
```
#### Running the original query using xinc:node-expand
**Not cached - 6 seconds!!**
```
<qm:elapsed-time>PT6.05373S</qm:elapsed-time>
```
**Cached - 2 seconds**
```
<qm:elapsed-time>PT2.154288S</qm:elapsed-time>
```

With our new optimised query we can see the time is much reduced below. This is a Mark Logic extension so we can't really do much about the performance of this. However it is interesting to see how much additional time this adds to the processing even for a fully optimised query.
```
<qm:elapsed-time>PT0.351562S</qm:elapsed-time>
```
From the above it is easy to see the majority of the query is spent in the xinc:node-expand function but we have increased the overall performance dramatically.

#### Conclusion
Even what is deemed to be the simplest of xquery/xpath expressions might be inefficient. Mark Logic won't tell you how to fix your xquery/xpath but it will provide insight into whether your query is utilizing indexes and how it is actually running.

#### Useful Links

* [http://developer.marklogic.com/blog/learning-with-query-trace](http://developer.marklogic.com/blog/learning-with-query-trace)
* [http://developer.marklogic.com/blog/how-to-write-fast-queries](http://developer.marklogic.com/blog/how-to-write-fast-queries)
* [http://developer.marklogic.com/pubs/4.2/apidocs/Ext-4.html](http://developer.marklogic.com/pubs/4.2/apidocs/Ext-4.html)
* [http://developer.marklogic.com/pubs/4.2/apidocs/Ext-4.html#xdmp:estimate](http://developer.marklogic.com/pubs/4.2/apidocs/Ext-4.html#xdmp:estimate)
* [http://xqzone.marklogic.com/pubs/3.2/apidocs/Extension.html#query-meters](http://xqzone.marklogic.com/pubs/3.2/apidocs/Extension.html#query-meters)
* [http://developer.marklogic.com/blog/xquery-coding-guidelines](http://developer.marklogic.com/blog/xquery-coding-guidelines)

