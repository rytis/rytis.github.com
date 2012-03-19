---
layout: post
title: "AJAX and TastyPie   check if a user has authenticated"
category: Django, Programming
tags: [django, tastypie]
---
{% include JB/setup %}

If you're using built in [TastyPie](https://github.com/toastdriven/django-tastypie) user authentication/authorization (HTTP Basic Auth) you might run into a problem with AJAX requests. Here's a typical situation:

* Your main web application is a Django application
* It requires users to authenticate
* The application also exposes some functionality via (TastyPie) API, which requires user authentication
* Your pages uses JavaScript to access this API

When users log on to the Django web application, they get redirected to the standard Dajngo auth pages, where they submit a login form with their credentials. If these are valid, a response is sent back with a session cookie. On Django side, this session cookie corresponds to a datastructure, which indicates that a user has already authenticated.

Now let's suppose an AJAX call is made to the API. TastyPie expects HTTP Basic Auth with user credentials, but they are not available. The only indication that a user has authenticated is the session cookie issued earlier. At this point a nasty browser pop up shows up and asks for user authentication details... So how do you tell TastyPie that the user has already authenticated?

One of the ways is to subclass the existing authentication class and attempt to retrieve user details based on the session data, luckily AJAX calls do include all session data, including cookies:

{% highlight python %}
class MyBasicAuthentication(BasicAuthentication):
    def __init__(self, *args, **kwargs):
        super(MyBasicAuthentication, self).__init__(*args, **kwargs)
 
    def is_authenticated(self, request, **kwargs):
        from django.contrib.sessions.models import Session
        if 'sessionid' in request.COOKIES:
            s = Session.objects.get(pk=request.COOKIES['sessionid'])
            if '_auth_user_id' in s.get_decoded():
                u = User.objects.get(id=s.get_decoded()['_auth_user_id'])
                request.user = u
                return True
        return super(MyBasicAuthentication, self).is_authenticated(request, **kwargs)

{% endhighlight %}

Here's what this snippet is doing:

* We check if there's a sessionid cookie
* If it is available, let's retrieve session data
* Once user has authenticated, a special session entry (`_auth_user_id`) is created by Django
* So we check if this entry is present
* If it is, then the user has authenticated successfully. We set the user object and return True
* For everything else we fall back to the default authentication implementation
