---
layout: post
title: "Evaluating Mark Logic XQuery Performance"
date: 2012-02-12 19:36
comments: true
categories: [XQuery, MarkLogic]
published: false 
---

I've recently been doing some work building RESTFul API's backed by a Mark Logic XML Content Store utilising XQuery for document retrieval. This post details the steps involved in tuning what was deemed to be the most simplest of queries for optimum performance using some useful mark logic extensions for profiling.

Original Query
--------------
XQuery for looking up documents based on the value of a given attribute in the xml using XPath. (What could be simpler!)
{% gist 1810958 document_by_url.xqy %}


{% gist 1810958 document_by_url_with_xdmp_query_meters.xqy %}
{% gist 1810958 result_of_xdmp_query_meters.xml %}
