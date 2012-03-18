---
layout: post
title: "Updated Exctractor code on sourceforge.net"
category: exctractor
tags: [documentation, python, regexp]
---
{% include JB/setup %}

I’ve just received an email from someone who was trying to use the [exctractor](http://exctractor.sourceforge.net/):

> I really like the idea behind your exctractor utility. But the whole point of me searching for a tool like this is that I'm having trouble writing the regexps for the parsing myself...
> 
> The exctractor.xml configuration example is painfully abscent of example regexp :)

I totally agree with this and I think it is a fair comment. Most of us assume that something that's obvious to us is automatically also known to others, which isn't necessarily always true. This is a known phenomena in psychology and it means that we often perform worse at documenting topics that we know a lot about.

So I added some sample Java exceptions and the corresponding Exctractor rules that catches them. I've also updated the default configuration file with some simple examples of regular expression rules syntax. Hopefully this will serve as an example and will make the initial configuration less painful.

Obviously, the regular expressions are quite difficult to master and there are number of books that explain them in great detail. The classic text on this subject is [Mastering Regular Expressions](http://www.amazon.com/gp/product/0596528124?ie=UTF8&tag=sysadminpy-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0596528124) by Jeffrey E F Friedl. For a quick start you could refer to the Python [re module](http://docs.python.org/library/re.html#regular-expression-syntax) documentation or visit AddedBytes that have a nice [Python regular expression cheat sheet](http://www.addedbytes.com/cheat-sheets/regular-expressions-cheat-sheet/) made available for you to use.
