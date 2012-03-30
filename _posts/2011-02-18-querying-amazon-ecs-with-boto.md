---
layout: post
title: "Querying Amazon ECS with Boto"
category: programming
tags: [amazon, aws, boto]
---
{% include JB/setup %}


Just like everyone else [who's ever written a book](http://ejohn.org/blog/programming-book-profits/), I'm really curious how it's doing. Unlike a blog where I can get statistics almost instantaneously it may take up to 3-4 months for the next report to arrive. Although Apress have always been responsive and helpful it's just the way things are.

Like John is suggesting I was checking Amazon every now and then and was trying to make some sense of the magic .sales rank. number. Granted, System Administration with Python topic interests only a relatively small number of people. Someone said that to me (not exact quote): "You take all geeks, from this set you take those that use Python, then strike off those that are not Linux system administrators and what's left is your audience".

Being a rather realistic person I did not expect sales to sky rocket, yet it is still quite interesting to see how the things are progressing.

So I knocked up a simple script that queried Amazon ECS (e-Commerce Service aka the Product Advertising API) for the sales rank fugure and recorded it in a Round Robin Database using RRDTool. I used [Boto](http://code.google.com/p/boto/) library, but unfortunately it didn.t have support for ECS at the time, so I only used it to setup a session, ie login to Amazon services.

Time passed and now Boto has some basic support of ECS. Although it is not (yet) documented on their website, the basic functionality is there and is relatively simple to use. However you need to use the latest code from GitHub if you want to get the examples below working properly.

The example script below queries Amazon ECS and retrieves a sales rank figure for a specific item. If you call the script with a locale ID (such as 'us', 'uk', etc) as a parameter it will show sales rank in that particular country. Otherwise it.ll show sales rank figures on all Amazon websites.

{% highlight python %}

#!/usr/bin/env python
 
import time
import urllib
import boto
import sys
from boto.ecs import ECSConnection
 
ECS_HOSTS = { 'us': 'ecs.amazonaws.com',
              'uk': 'ecs.amazonaws.co.uk',
              'jp': 'ecs.amazonaws.jp',
              'fr': 'ecs.amazonaws.fr',
              'de': 'ecs.amazonaws.de',
              'ca': 'ecs.amazonaws.ca',
            }
 
class ExtendedECSConnection(ECSConnection):
 
    def __init__(self, aws_access_key_id=None, aws_secret_access_key=None, host='ecs.amazonaws.com'):
        super(ExtendedECSConnection, self).__init__(aws_access_key_id=aws_access_key_id,
                                                    aws_secret_access_key=aws_secret_access_key,
                                                    host=host)
 
    def item_lookup(self, item_id, **params):
        params["ItemId"] = item_id
        return self.get_response('ItemLookup', params)
 
def main():
    hosts = []
    if len(sys.argv) > 1:
        hosts.append(ECS_HOSTS[sys.argv[1]])
    else:
        hosts = ECS_HOSTS.values()
    for host in hosts:
        conn = ExtendedECSConnection(host=host)
        res = conn.item_lookup(1430226056, ResponseGroup="SalesRank")
        sales_rank = [item.get('SalesRank') for item in res][0]
        print "%s:%s" % (host, sales_rank)
 
if __name__ == '__main__':
    main()

{% endhighlight %}

Here, I'm extending the ECSConnection class and introducing a new method for retrieving just a single item. Since I'm only interested in the sales rank figure, I'm requesting the response to be in the **SalesRank** group. Amazon ECS has multiple response groups, you can find more information about the API and request parameters on their [API documentation](http://docs.amazonwebservices.com/AWSECommerceService/2010-11-01/DG/) web page.

<img src="/assets/images/monthly_rank.png" width="600" />

<img src="/assets/images/yearly_rank.png" width="600" />

&nbsp;
As you can see from these graphs the fluctuations of the sales rank figures aren't really that "crazy" after all. I'm guessing the sharp drops are simply an indication of another sale made through the Amazon website.

**UPDATE:** An interesting [analysis of the Amazon sales rank](http://www.fonerbooks.com/surfing.htm) supports the graphs above. I suspect the sharp drops to be the actual sales and therefore a sales rank that falls in range 100k to 500k actually reflects 1-5 sales per week.

**UPDATE 2:** The graphs used to be "live" and re-generated every hour, but since I moved the site to GitHub pages, the live update is no longer maintained.

