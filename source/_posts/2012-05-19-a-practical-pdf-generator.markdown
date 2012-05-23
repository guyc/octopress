---
layout: post
title: "FOML: Format Object Markup Language"
date: 2012-05-19 21:41
comments: true
categories: FOML, PHP
---

In Search of a Practical PDF Framework
----------

So here I am in that dark place again; I need to produce pretty PDF reports from
a PHP web app, with headers and footers and tables that wrap and paginate nicely.
Every time I get here I look for an elegant
solution, but frankly all of them ended up sucking.

My last approach was built around [TCPDF](http://www.tcpdf.org).  It's an excellent
low-level PDF toolkit, but any non-trivial document generation code always ends
up ugly as hell. How is it that 25 years after PostScript promised us "device independent"
document layout, I am still dicking around writing layout and pagination logic
for PDF reports?

XSL-FO FTW!
-------

[XSL-FO](http://en.wikipedia.org/wiki/XSL_Formatting_Objects) is
an XML markup language that lets you specify
document layout in a sane way, and [Apache's FOP](xmlgraphics.apache.org/fop/)
will render XLS-FO documents into PDF's.  This is really cool - these _are_ the droids
I've been looking for; they take care of all of the heavy lifting, so 
we just need to build a PHP interface.

But there's a wee problem.  After years in the wilderness writing PHP code to emit
well formed XHTML, then working on a few Rails projects using [HAML](http://haml.info)...
well what has been seen cannot be unseen.  Now I use [PHAML](http://phaml.sourceforge.net/) in
PHP for any non-trivial XHTML generation.  HAML lets you 
neatly [isolate](http://en.wikipedia.org/wiki/Separation_of_concerns) your layout
logic from your application logic, and leaves your code and your breath minty fresh.
So I really can't get all excited about writing PHP to spew forth well-formed XLS-FO.

FOML - Formatting Object Markup Language
----

{% pullquote %}
So, duh, connect the dots.  Steal Hampton Catlin's goddam brilliant insight that became HAML
and apply it to XSL-FO.  {" FOML is to XSL-FO as HAML is to HTML. "} Consider this "hello world" in XSL-FO:
{% endpullquote %} 

{% codeblock XSL-FO Hello World %}
<?xml version="1.0" encoding="iso-8859-1"?>

<fo:root xmlns:fo="http://www.w3.org/1999/XSL/Format">
  <fo:layout-master-set>
    <fo:simple-page-master master-name="my-page">
      <fo:region-body margin="1in"/>
    </fo:simple-page-master>
  </fo:layout-master-set>

  <fo:page-sequence master-reference="my-page">
    <fo:flow flow-name="xsl-region-body">
      <fo:block>Hello, world!</fo:block>
    </fo:flow>
  </fo:page-sequence>
</fo:root>
{% endcodeblock %}

Here's the same thing expressed in FOML:

{% codeblock FOML Hello World %}
!!!
%root(xmlns:fo="http://www.w3.org/1999/XSL/Format")
  %layout-master-set
    %simple-page-master(master-name="my-page")
      %region-body(margin="1in")

  %page-sequence(master-reference="my-page")
    %flow(flow-name="xsl-region-body")
      %block 
        Hello, world!

{% endcodeblock %}

Unless you work in a Hello World factory (sweet job!) 
in practice the layout-master-set block gets more complex, and 
is likely to be common to all of the documents in a project, so lets use
an :include filter to pull in a shared layout-master-set partial.
HAML doesn't have an include filter, FOML does.

{% codeblock FOML Hello World %}
!!!
%root(xmlns:fo="http://www.w3.org/1999/XSL/Format")
  :include('A4-Layout.foml')

  %page-sequence(master-reference="my-page")
    %flow(flow-name="xsl-region-body")
      %block 
        Hello, world!

{% endcodeblock %}

FOML With Inline Code
----

Concise syntax and automatic closing of tags is nice, but it
is not why we are here.  FOML, like HAML, supports inline code.  The following FOML document
iterates through an array of records and generates a nicely wrapped and paginated
3-column multi-page table.

{% codeblock FOML Inline Code Example %}
!!!
-# Parameters
-#   $record:  array of Record objects.

%root(xmlns:fo="http://www.w3.org/1999/XSL/Format")
  :include('A4-Layout.foml')

  %page-sequence( master-reference="my-page")
    %flow(flow-name="xsl-region-body")
      %block

        / table start
        %table(table-layout="fixed" width="100%" border-collapse="separate")
          %table-column(column-width="50mm")
          %table-column(column-width="50mm")
          %table-column(column-width="50mm")
          %table-body
            - foreach ($records as $record)
              %table-row
                %table-cell
                  %block 
                    = $record->name
                %table-cell
                  %block
                    = $record->address
                %table-cell
                  %block
                    = $record->telephone

{% endcodeblock %}

And here's how you would turn that into a PDF from PHP.

{% codeblock FOML from PHP %}
<?php
  $records = AddressBook::GetAll();
  FOML::RenderToPdf("TableReport.foml", array('records'=>$records));
?>
{% endcodeblock %}

Easy as, right?

PHP Implementation
------------------

A PHP library implementing FOML is [available on github](https://github.com/guyc/FOML).  The
library requires Apache FOP, which in turn requires Java.
It is complete enough to be useful, but is not feature-complete.
The documentation describes the features that are implemented.

Also note that the overhead of starting Java for Apache FOP is non-trivial.
On my machines it adds >2 seconds, so even a trivial PDF report takes
that long to generate.  At some point I plan to try running FOP as a daemon
to eliminate the startup latency, but for now expect XLS-FO to PDF translation
to take a few seconds.








