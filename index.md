---
layout: page
---
{% include JB/setup %}

<div class="hero-unit">
    <div class="row">
        <div class="span8">
            <p>This is a website for the Pro Python System Administration book, written by Rytis Sileika and published by <a href="http://www.apress.com/">Apress</a>. Some of the projects described in the book are avialable on <a href="https://www.github.com/rytis/">GitHub</a>.</p>
            <p><a href="http://www.amazon.com/gp/product/1430226056?ie=UTF8&tag=sysadminpy-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1430226056" class="btn btn-primary btn-large"><i class="icon-shopping-cart icon-white">&nbsp;</i>&nbsp;Buy from Amazon (US) &raquo;</a>&nbsp;<a href="http://www.amazon.co.uk/gp/product/1430226056?ie=UTF8&tag=sysadminpy-21&linkCode=as2&camp=1634&creative=19450&creativeASIN=1430226056" class="btn btn-primary btn-large"><i class="icon-shopping-cart icon-white">&nbsp;</i>&nbsp;Buy from Amazon (UK) &raquo;</a></p>
        <p><a href="http://books.google.co.uk/books?id=Jxij_jnCS4AC&lpg=PR2&dq=1-4302-2605-6&pg=PR2#v=onepage&q&f=false" target="_blank">Preview on Google Books &rarr;</a></p>
        </div>
        <div class="span2">
            <p><img src="/assets/images/book-cover.png" width="150"></p>
        </div>
    </div>
</div>

## Latest blog post

{% assign latest_post = site.posts.first %}

### <a href="{{ BASE_PATH }}{{ latest_post.url }}">{{ latest_post.title }}</a>
<h6>{{ latest_post.date | date_to_string }}</h6>

{{ latest_post.content }}

<hr/>

<a href="{{ BASE_PATH }}{{ site.JB.archive_path }}">Jump to Blog Archive &rarr;</a>
