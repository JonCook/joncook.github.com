---
layout: post
title: "XML Processing with Scala"
date: 2013-11-03 08:33
comments: true
categories: [Scala, XML, Functional]
---

Given scala's built-in support for XML and its more concise syntax for dealing with iterables and collections I was interested to see what some of our java xml parsing code written using Dom4j could look like in the Scala world.

### Background
A bit of brief background, the idea is we have a number of assets, an asset representing for example a story, index page, picture gallery, page with media. Each Asset has it's own xml representation and is constructed using common snippets such as item-meta, page-options, media etc. See story.xml.

Our requirement, is to take this xml which is our own internal representation which we use to repesent assets in our content store and essentially transform this into our object model and serve this up as json or xml. You may ask why but isolating our internal format means we are free to change this without effecting the external representations of our assets and it also means we can customise the output and the format of the output easily.

Xml processing in java is typically done in sax or dom using a library like dom4J, we opted for the readability and ease of use the Dom over SAX and used something loosely based on the Strategy design pattern where we have a ParserFactory, this allocates the correct type of asset parser based on the type of asset parses the xml and creates our java object model. Xml and Json generation is handled by jaxb and jackson.
<!-- more -->

### POJO's 

The recommendation looking at the scala documentation is to implement a toXML method and a corresponding fromXML method from within side a 
companion object which takes care of creating said object. e.g) A Story POJO which extends an Asset class. 

{% codeblock lang:scala %}
package com.cookybear.content.asset

import scala.xml.NodeSeq

class Story(
  itemMeta: ItemMeta,
  pageOptions: PageOptions,
  byline: Byline,
  body: NodeSeq,
  media: Media,
  relatedGroups: RelatedGroups) extends Asset(itemMeta, pageOptions) {

  override def toXML =
    <result>
      { super.toXML.child }
      { byline.toXML }
      { body }
      { media.toXML }
      { relatedGroups.toXML }
    </result>
}

object Story {
  def fromXML(node: scala.xml.NodeSeq): Story =
    new Story(
      ItemMeta.fromXML((node \ "itemMeta")),
      PageOptions.fromXML((node \ "pageOptions")),
      Byline.fromXML((node \ "byline")),
      (node \ "body"),
      Media.fromXML((node \ "media")),
      RelatedGroups.fromXML(node \ "relatedGroups"))
}
{% endcodeblock %}

###  Parsing and iterating over elements 

Parsing and iterating over elements is much neater and concise than the java equivalent e.g) A byline is made up a name, a title and a list of 
Person objects.

{% codeblock lang:scala %}
package com.cookybear.content.asset

case class Byline(val name: String, val title: String, val persons: List[Person]) {

  def toXML =
    <byline name={ name } title={ title }>
      {
        if (!persons.isEmpty)
          <persons>{
            for (person <- persons) yield person.toXML
          }</persons>
      }
    </byline>

}

object Byline {
  def fromXML(node: scala.xml.NodeSeq): Byline =
    new Byline(
      name = (node \ "@name").text,
      title = (node \ "@title").text,
      List[Person]((node \ "person").toList map { s => Person.fromXML(s) }: _*))

}
{% endcodeblock %}

In my opinion this is where you start to see the real power of Scala by calling .toList on the sequence of person nodes you can then map this to a list of people. Person will have its own corresponding toXML implementation as well. We remove those horrible null checks and cumbersom for loops and my favourite removing all the static xPath constants. No need to worry about name spaces either.

The equivalent in java for parsing the byline and person elements which doesn't include the pojo for byline either. Much more cumbersom and messy and you've got no feel for what the corresponding structure of the xml will look like when we deserialize a byline. 

{% codeblock lang:java %}
public Byline parseByline(Element bylineElement) { 
    Byline byline = null; 
    if (bylineElement != null) { 
        byline = new Byline(); 
        byline.setName(bylineElement.attributeValue("name")); 
        byline.setTitle(bylineElement.attributeValue("title")); 

        List<Element> persons =  BYLINE_PERSON_XPATH_SELECTOR.selectNodes(bylineElement); 
        for (Element element : persons) { 
           Person person = parsePerson(element); 
           byline.getPersons().add(person); 
        } 
    } 
    return byline; 
} 

private Person parsePerson(Element element) { 
    Person person = new Person(); 
    person.setThumbnail(getAbsoluteImageHref(element.attributeValue("thumbnail"))); 
    person.setFunction(BYLINE_PERSON_FUNCTION_XPATH_SELECTOR.valueOf(element)); 

    person.setName(BYLINE_PERSON_NAME_XPATH_SELECTOR.valueOf(element)); 
    return person; 
} 

{% endcodeblock %}  

### Plugging it together Using scala's XML Pattern Matching 

We are able to eliminate the existing ParserFactory class and have an AssetFactory which allocates the right type of Asset e.g) 

{% codeblock lang:scala %} 
package com.cookybear.content.asset.factory

import com.cookybear.content.asset.Asset
import com.cookybear.content.asset.Story

object AssetFactory {

  def factory(node: scala.xml.Node): Asset = {
    val trimmedNode = scala.xml.Utility.trim(node)

    trimmedNode match {
      case <story>{ children @ _* }</story> => Story.fromXML(trimmedNode)
    }

  }

}
{% endcodeblock %}


### Testability
I think the code is more testable now as well. I prefer the readability of these type of tests using something like ScalaTest. I created my own convenience trait XmlDataSpec which is essentially a way to minimise the number of mixins used in our tests. FixtureTestUtils gives us a way to load in xml fixtures. You can load in snippets from file or inline the xml element you wish to test and it all just seems more natural, readable and less verbose than the java equivalent. e.g) 

{% codeblock lang:scala %}
package com.cookybear.content.asset

import com.cookybear.content.XmlDataSpec
import org.junit.runner.RunWith
import org.scalatest.junit.JUnitRunner

@RunWith(classOf[JUnitRunner])
class BylineSpec extends XmlDataSpec {

  describe("A Byline with a list of authors") {
    val byline = Byline.fromXML(xmlFixture("/xml/byline.xml"))

    it("should have name and title") {
      assert(byline.name === "By AJP Taylor & Adam Brookes & Alan Hansen")
      assert(byline.title === "Historian")
    }

    it("should contain a list of people") {
      byline.persons must have length (3)
    }

  }

}

{% endcodeblock %}

###  What I like (lots\!) 

* No need for separate parser classes which contain huge amounts of static constants declaring xpath snippets. 
* You can see in one place the xml representation of the class and also the parsing code to create the object. 
* It is easy to see what for example a Story is made up of, in this case, itemMeta, PageOptions, Byline, Media and Related Groups 
* Nulls are taken care of and iterating over nodes is much neater. 
* Generating xml doesn't require any frameworks, I've been frustrated by the inflexibility of jax-b particularly around handling empty elements and 
also having to define lots of custom adapters which are a disaster, not to mention ending up with lots of annotations in your pojos.

### Improvements 

* I think there is probably a nicer way to handle conditional elements using None and Some  {% codeblock lang:scala %}{if (!title.isEmpty) <title>{title}</title>}{% endcodeblock %} 
* Inline xml in pojos is ok for small snippets but I could see this becoming messy if care wasn't taken 

### The code 

I've put the sample code on github, [here](https://github.com/JonCook/scala-xml-parsing-example). You'll need SBT installed and once you have checked it out simple run sbt test to see the unit tests. In addition to this Main.sala is a simple test harness for processing a story. Please remember I'm not a scala expert, I'm pretty sure there is a way to improve the double .toList calls in Media.scala using zip and also some people may suggest that some of the functions inside .toList aren't readable but I think they are if you are familiar with scala.

Gists are available [here](https://gist.github.com/JonCook/8397116) for the code here.

### Conclusion
I've seen a lot of complaints about the current implementation of scala.xml but for simple xml representations and parsing I think it works really well and is much more readible than the java equivalent. The performance is equal if not better than dom4j  
