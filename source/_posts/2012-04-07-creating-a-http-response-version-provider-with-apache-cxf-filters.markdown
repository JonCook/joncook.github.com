---
layout: post
title: "Creating a HTTP Response Version Provider with Apache CXF Filters"
date: 2012-04-07 22:16
comments: true
categories: [Apache CXF java Tomcat]
---

I've been using [Apache CXF](http://cxf.apache.org/) which has some nice support for building JSR-311 compliant JAX-RS Services and recently had a requirement for sending back a custom version header as part of our API Responses.

It is generally accepted that versioning APIs is a good thing and adopting the general contract of Major.Minor.Patch covers our needs, now we need to have a standard method of reporting and documenting the APIs and versions. Before we get into the ins and outs of how one you should version an API, we have a company wide recommendation of adding a custom response header to each response **X-API-Version: d.d.d** which I have to say I'm in favour of over embedding the version in the url and is also the method adopted by Sun’s Cloud API - which certainly was commonly held to be a benchmark implementation of REST.

Using [Apache CXF Filters](http://cxf.apache.org/docs/jax-rs-filters.html) led to this fairly elegant and simple solution which could be applied to add any type of additional headers but I found the documentation a little confusing so decided to write a post on it.
<!-- more -->

### Some background about applying response filters
Perhaps you want to do custom logging or additional processing but more interestingly the response handler implementation can optionally overwrite or modify the application Response or modify the output message the first of which we are particularly interested in.

By implementing The ResponseHandler Interface:
{% codeblock lang:java %}
import org.apache.cxf.jaxrs.ext.ResponseHandler;
{% endcodeblock %}

And overriding handleResponse:
{% codeblock lang:java %}
@Override
public Response handleResponse(Message message, OperationResourceInfo info, Response response) {
	return Response.ok().build();
}
{% endcodeblock %}


### Starting with a test
Starting off with the simplest scenario which for me is that the X-API-Version header should be present in the response. Normally I'd start with the assert statement and work backwards using my Eclipse shortcuts. I've skipped ahead a few steps here but hopefully you get the idea.

``` java
@Test
public void handleResponseShouldAddXAPIVersionHeader() throws Exception {
	HttpResponseVersionProvider versionProvider = new HttpResponseVersionProvider();

	Response versionedResponse = versionProvider.handleResponse(null, null, null);
	assertEquals("1.1.11", versionedResponse.getMetadata().get("X-API-Version").get(0));
}
```

#### Diving into our HttpResponseVersionProvider
So remember we are just doing the simplest thing possible to make the test pass which in this case is to add the new header to the final response.

``` java
@Override
public Response handleResponse(Message message, OperationResourceInfo info, Response response) {
	return Response.ok().header("X-API-Version", "1.1.11").build();
}
```
### Preserving the original response object
We obviously want to preserve the original response object and message body on the outgoing response so I add a test for that now introducing some Mocks to mock out the original response object.

``` java
	@Test
	public void handleResponseShouldMaintainOriginalResponseValuesAndAddXAPIVersionHeader() throws Exception {
		String responseEntity = "Response";
		when(response.getStatus()).thenReturn(200);
		when(response.getEntity()).thenReturn(responseEntity);
		when(response.getMetadata()).thenReturn(new MetadataMap<String, Object>());
		
		HttpResponseVersionProvider versionProvider = new HttpResponseVersionProvider(API_VERSION_NUMBER);
		
		Response versionedResponse = versionProvider.handleResponse(null, null, response);
		assertEquals(200, versionedResponse.getStatus());
		assertEquals(responseEntity, versionedResponse.getEntity());
		assertEquals(API_VERSION_NUMBER, versionedResponse.getMetadata().get(HttpResponseVersionProvider.VERSION_HEADER).get(0));
	}

```

#### Our final HttpResponseVersionProvider
[fromResponse](http://jackson.codehaus.org/javadoc/jax-rs/1.0/javax/ws/rs/core/Response.html#fromResponse%28javax.ws.rs.core.Response%29) performs a shallow copy of the existing Response.
``` java
	@Override
	public Response handleResponse(Message message, OperationResourceInfo info, Response response) {
		return Response.fromResponse(response).header(VERSION_HEADER, version).build();
	}
```

#### Refactor
You can see the tidied up finished test class and implementation here:

* [HttpResponseVersionProviderTest.java](https://gist.github.com/2332281)
* [HttpResponseVersionProvider.java](https://gist.github.com/2336443)

### Wiring it up
Plug in to your cxf jax-rs providers spring configuration along with any other providers using your maven project version number.
``` xml

<jaxrs:providers>
	<ref bean="jaxRsExceptionHandler"/>
    <ref bean="httpRequestMonitor"/>
    <ref bean="assetWhitelistFilter"/>
    <ref bean="httpResponseMonitor"/>
    <ref bean="httpResponseVersionProvider"/>
</jaxrs:providers>

<bean id="httpResponseVersionProvider" class="bbc.forge.dsp.jaxrs.HttpResponseVersionProvider">
	<constructor-arg value="${project.version}"></constructor-arg>
</bean>

```



