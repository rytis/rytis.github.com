---
layout: post
title: "A minimalistic REST client"
category: programming
tags: [rest, python]
---
{% include JB/setup %}

I needed a simple REST client to test a web service that I am developing with [Django](http://www.djangoproject.com/) and [TastyPie](http://django-tastypie.readthedocs.org/). Although TastyPie can easily handle a variety of formats (JSON, XML, HTML, ...) I'm primarily interested in JSON.

So I needed a simple client that would implement the four basic HTTP request types (GET, PUT, POST, DELETE) and would be simple enough to use with JSON data payload.

Essentially, I needed a simple wrapper around `httplib` that simplified the HTTP calls, and allowed me to do something along those lines:

{% highlight python %}

result = client.put('/api/resource/', data_json=json.dumps(request_data))
result = client.delete('/api/resource/')

{% endhighlight %}

You can find the source code of this library (which I named miniREST) on [GitHub](https://github.com/rytis/miniREST/). It is also available on [PyPi](http://pypi.python.org/pypi/miniREST/) and can be installed with [PIP](http://pypi.python.org/pypi/pip) package manager.

What I wanted to show you here is how to handle calls to object methods, that aren.t actually defined as class functions.

Let me explain what I mean. Going back to the REST client example, I might have defined a class and four methods implementing four different HTTP calls: GET, PUT, POST and DELETE. Maybe something like this:

{% highlight python %}

class RESTClient:
    def get(self, uri):
        result = httplib('GET', uri)
        # do some data parsing
        return result_data
 
    def put(self, uri, data):
        result = httplib('PUT', uri, body=data)
        # do some data parsing
        return result_data
 
    def post(self, uri, data):
        result = httplib('POST', uri, body=data)
        # do some data parsing
        return result_data
 
    def delete(self, uri):
        result = httplib('DELETE', uri)
        # do some data parsing
        return result_data
 
{% endhighlight %}

This would work, but it doesn.t feel quite right. You would be replicating effectively the same code four times. If you find a bug or want to make an improvement in the result data parsing you'll have to make changes in four different functions. This can lead to an inconsistent code behaviour if you forget to change one function or make a mistake while changing it.

My approach to this problem was to create a generic method that is able to handle all four HTTP requests. Yet I wanted to be able to call GET, PUT, POST and DELETE as object methods . I think it's more readable.

So I'm going to make use of the built-in method `__getattr__`. This method is called every time if an attribute is not found using the usual methods, for example by looking up the object's `__dict__` attribute, which contains all attributes of an object as a dictionary.

The `__getattr__` method should return either an attribute value or raise an AttributeError exception. This exception will be raised automatically if you try to reference a non-existant attribute and the `__getattr__` is not defined:

{% highlight python %}

>>> class C:
...  def __init__(self):
...   pass
...
>>> o = C()
>>> o.foo
Traceback (most recent call last):
  File "", line 1, in
AttributeError: C instance has no attribute 'foo'

{% endhighlight %}

However if the `__getattr__` is defined it will be called instead:

{% highlight python %}

>>> class C:
...  def __init__(self):
...   self.bar = 1
...  def __getattr__(self, attribute):
...   print "You're trying to read %s, but it's not initialised yet" % attribute
...   return 0
...
>>> o = C()
>>> r = o.bar
>>> print r
1
>>> r = o.foo
You're trying to read foo, but it's not initialised yet
>>> print r
0

{% endhighlight %}

OK, we're getting there now. But there is a slight problem. The `__getattr__` returns a computed value of an attribute. It's easy for variables, but we want to reference a function, so `__getattr__` needs to return a reference to a method and not the computed value.

This is easy, but our function would not know under what name it has been called:

{% highlight python %}

>>> class C:
...  def _my_func(self):
...   print "in a function"
...  def __getattr__(self, attr):
...   return self._my_func
...
>>> o = C()
>>> o.test()
in a function
>>> o.foo()
in a function

{% endhighlight %}

Because we return a reference to a function (as opposed to actually calling it and returning a result) we can.t pass any arguments to it that would tell under what name it's been called.

One of the easiest ways to solve this is to store the name as an object attribute and then look it up from the function:

{% highlight python %}

>>> class C:
...  def __init__(self):
...   self.func_name = None
...  def _my_func(self):
...   print "Function called as '%s'" % self.func_name
...  def __getattr__(self, attr):
...   self.func_name = attr
...   return self._my_func
...
>>> o = C()
>>> o.foo()
Function called as 'foo'
>>> o.bar()
Function called as 'bar'

{% endhighlight %}

And this is exactly the behaviour I was after. Now it's an exercise to the reader . can you think of any problems that this approach may cause?


