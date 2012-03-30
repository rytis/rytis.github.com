---
layout: page
---
{% include JB/setup %}

<div class="hero-unit">
    <div class="row">
        <div class="span8">
            <p>This is a website for the Pro Python System Administration book, written by Rytis Sileika and published by <a href="http://www.apress.com/">Apress</a>. Some of the projects described in the book are avialable on <a href="https://www.github.com/rytis/">GitHub</a>.</p>
            <p><a href="http://www.amazon.com/gp/product/1430226056?ie=UTF8&tag=sysadminpy-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1430226056" class="btn btn-primary btn-large">Buy from Amazon (US) &raquo;</a>&nbsp;<a href="http://www.amazon.co.uk/gp/product/1430226056?ie=UTF8&tag=sysadminpy-21&linkCode=as2&camp=1634&creative=19450&creativeASIN=1430226056" class="btn btn-primary btn-large">Buy from Amazon (UK) &raquo;</a></p>
        </div>
        <div class="span2">
            <p><img src="/assets/images/book-cover.png" width="150"></p>
            <p><script type="text/javascript" src="http://books.google.com/books/previewlib.js"></script>
                <script type="text/javascript">
                    GBS_insertPreviewButtonPopup('ISBN:1-4302-2605-6');
                </script>
            </p>
        </div>
    </div>
</div>

## Latest blog post

{% assign latest_post = site.posts.first %}

### <a href="{{ BASE_PATH }}{{ latest_post.url }}">{{ latest_post.title }}</a>
<h6>{{ latest_post.date | date_to_string }}</h6>

{{ latest_post.content }}
