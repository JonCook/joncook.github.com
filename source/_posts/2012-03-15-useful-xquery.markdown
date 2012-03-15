---
layout: post
title: "Useful XQuery"
date: 2012-03-15 13:58
comments: true
categories: [XQuery, MarkLogic]
---

As I mentioned in my previous [blog post](http://joncook.github.com/blog/2012/02/12/evaluating-mark-logic-xquery-performance/) I've recently been doing some work building RESTFul API's backed by a Mark Logic XML Content Store utilising XQuery for document retrieval which has led me come up with the following useful snippets of XQuery that I thought I would share. They could easily be altered to work with slightly different requirements.

### List all the document uris in database
--------------
{% gist 2039314 list_all_document_uris.xqy %}

### List all the document uris in database based on some criteria
--------------
In this case we are restricting the documents that exist in the /sitemap/ directory and end with sitemap.xml and do not contain archive in the uri
{% gist 2039314 list_document_uris_where_uri_matches_criteria.xqy %}

### Delete all the document in the database
--------------
{% gist 2039314 delete_all_documents.xqy %}

### Delete all the document in database based on some criteria
--------------
In this case we are deleting documents that are in the /content/ directory and whose uri ends with NEWSML.xml
{% gist 2039314 delete_documents_where_uri_matches_criteria.xqy %}

### Query a document by an element with a certain value
--------------
This could be easily altered to match on any element or attribute in your xml
{% gist 2039314 get_document_where_element_matches_value.xqy %}

Actually a much more efficient way of doing the same is:
{% gist 2039314 more_efficient_get_document_where_element_matches_value.xqy %}
<!-- more -->

Slightly more advanced...

### Retrieving document sizes
--------------
Very useful for getting a view of the size of your documents
{% gist 2039314 document_sizes_in_kb.xqy %}

This is how the results are returned
{% gist 2039314 results_of_document_sizes_in_kb.xml %}

### Adding a depth attribute to your xml documents
--------------
Why is this useful I hear you say, demonstrates how to use xdmp:node-replace and xdmp:node-insert-child
{% gist 2039314 adding_a_depth_attribute_to_elements.xqy %}