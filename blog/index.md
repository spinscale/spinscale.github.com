---
layout: page
title: Toying with technology
tagline: Unimportant views on technology
---
{% include JB/setup %}

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



## Presentations

### 2012
#### [Elasticsearch - Search made easy for (web) developers](http://spinscale.github.com/elasticsearch/2012-03-jugm.html)
A long introduction into elasticsearch, made with [reveal.js](https://github.com/hakimel/reveal.js). Held at the [Java User Group Munich](http://www.jugm.de) and the [Berlin Expert Days Conference](http://www.bedcon.org)

### 2011
#### [Elasticsearch - Introduction](http://spinscale.github.com/elasticsearch/es-introduction.html)
A ten minute introduction at the Webmontag Munich November, hold in front of a mainly non-technical crowd.

#### [Play Framework Introduction](http://spinscale.github.com/play-introduction.html)
An introductory presentation about the play framework held in October 2011 at [TNG technologies](http://www.tngtech.com) in Munich. Thanks to them for giving me the opportunity to talk.

#### [Play Framework - A look inside the machine room](http://spinscale.github.com/play-advanced-concepts.html)
This presentation tries to go further than the standard introductory presentations and takes a look in the inner works of the play framework. This presentation was held at the Play!Ground event in Rotterdam in September. Thanks to Peter Hilton and [Lunatech](http://www.lunatech-research.com/) for inviting me!

### 2010
#### [Play Framework Introduction](http://www.slideshare.net/areelsen/introduction-playframework)
An introductory presentation about the play framework held in June 2010 at the Java User Group Munich.



## Projects

### Elasticsearch FST suggester plugin

An implementation of the Lucene FST Suggester in elasticsearch. More at the [Project Page](https://github.com/spinscale/elasticsearch-suggest-plugin).



## About

<div>
<img src="http://spinscale.github.com/images/alr.jpg" alt="Me, myself and I" id="pic" style="float:left;padding-right: 2em">
I am a Software Engineer living in Munich, author of the [Play Framework Cookbook](http://www.packtpub.com/play-framework-cookbook/book) (check the <a href="https://github.com/spinscale/play-cookbook">samples</a>). I do like hacking on different stuff. I feel comfortable with modern tools like Play Framework, MongoDB, Elasticsearch, Web, Linux, Concurrency and JavaScript. On bad days I try out GUI stuff. I keep losing.<br />If you want me to give a talk, contact me.
</div>
<div style="clear: both"></div>



