---
category: big-data
tags:
  - big data
  - jsv
  - python
title: Introduction to jsv
comments: true
---

With my new serialization format, jsv (for json separated values) I aim to create a format that has all the expressiveness of the popular json lines format, but with significant compression.

Motivation
----------

There has long been a need for a text-based serialization format for large numbers of records in which most or all of the records are of a similar format. Historically, csv (comma separated values) has dominated, because it is simple to understand and implement. However it has a number of serious flaws:

* It is not well defined. Other than using the endline character to delimit records, and the comma character to delimit fields, there is no universal standard for encoding.
* It can be a leaky abstraction if the keys (field names) are not in the file. Even if the keys are in the file, type information must be inferred from the field entries.
* It is restricted to flat dictionaries. Nesting must be hacked; for example, by making one of the fields a delimited string with a different delimiter.
* Arrays are not supported; they must also be hacked, usually by naming fields things like ``array_1``, ``array_2``, etc.

These failings of universality and expressiveness have been offset by simplicity and readability. Formats such as xml and json were introduced to add both universal parsing standards and add provide a significant increase in expressiveness. Xml unfortunately has a confused notion of attributes vs. child elements, and no clear notion of an array. With the introduction of json, however, the world finally had a serialization format that had just the right pieces: dictionaries, arrays, primitives, and no more.

Unfortunately, both json and xml came with considerable overhead, owing to the fact that both formats are designed to represent a *single* complex document, not a series of similar documents. They took the most naive approach to preventing leaky abstractions: the include all structural metadata in the document itself. Both formats have ways to define schemas, but these are not used for compression; they instead enforce constraints on document structure.

Jsv aims to enhance the usefullness of json by providing a way to strip redudant metadata from each record, and store it instead in a single directive that is anywhere in the file or stream--as long as it occurs before it is used for parsing. All other parsing rules follow json convention. As a result, jsv is a strict superset of json lines, and writing a parser requires only minor modifications to existing json parsers.

Design Criteria
---------------

The format is guided by the following priciples:

* Text-based
* Compress the json lines format by eliminating obviously redundant metadata. However, I am not optimizing on the overall compression factor. The aim is to acheive reasonable compression while preserving as much of json's other characteristics as possible.
* Should be readable to someone who knows the struture of the records.
* Should use json standards for everything other than metadata. For example, all primitives should be parsed according to json rules.
* Should require very little work to define the metadata. Most of the metadata should be inferred as records are being read or created.

The last requirement is as much a requirement of implementation libraries as it is of the format itself.

Examples
--------

### flat dictionary

Let's start with a flat dictionary of the kind that would easily be encoded with csv:

{% highlight json %}
{"key_1": "record_1", "key_2": 1234, "key_3", true}
{"key_1": "record_2", "key_2": 5678, "key_3", false}
{% endhighlight %}

This would convert to the following:

{% highlight json %}
#A{"key_1","key_2","key_3"}
A{:"record_1",1234,true}
A{:"record_2",5678,false}
{% endhighlight %}

Let's cover the basics. The first line defines (#) a new keymap, with the tag "A". Tags in jsv are just capital letters in sequence; essentially base 26 with A = 0, B = 1, etc. The values are represented as a list of json primitives, but they are enclosed in curly braces to indicate that they are part of a dictionary, not an array.

There is one slight anomaly in the primitives, which is that the string primitives are prepended with a colon (:). This is to distinguish string primitive from keys. More on this later.
