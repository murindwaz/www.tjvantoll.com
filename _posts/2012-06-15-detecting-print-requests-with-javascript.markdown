---
layout: post
title: Detecting Print Requests with JavaScript
comments: true
---

CSS has a well supported mechanism for applying changes only when the user is printing a document, [print stylesheets](http://coding.smashingmagazine.com/2011/11/24/how-to-set-up-a-print-style-sheet/).  They allow you to alter the presentation of a web page for the printer by applying rules that will only be interpreted for printing.  This is great for common tasks like hiding non-essential content, using more print friendly typography, and adjusting the layout to better suit the size and shape of paper.

Print stylesheets are great for making presentational changes for printing, but sometimes you need the full power of JavaScript.  And in order to do respond to print requests in JavaScript you need the browser to notify you that a print request occurred.

<!--more-->

## onbeforeprint and onafterprint

IE5+ fires `onbeforeprint` and `onafterprint` events before and after the user requests the page to be printed.

<pre class="language-javascript"><code class="language-javascript">window.onbeforeprint = function() {
    console.log('This will be called before the user prints.');
};
window.onafterprint = function() {
    console.log('This will be called after the user prints');   
};
</code></pre>

These events are not part of any specification but they are very convenient.  Because of this [Firefox added support for both events in version 6](https://developer.mozilla.org/en/DOM/window.onbeforeprint#Browser_compatibility).  However, WebKit and Opera do not support the events.  Therefore, for cross browser compatibility these events aren't going to cut it.

## WebKit's Solution

WebKit has a bug (#[19937](https://bugs.webkit.org/show_bug.cgi?id=19937)) out there to implement these events, but progress has stopped because the implementation of another API made this functionality possible already - `window.matchMedia`.

## window.matchMedia

The `window.matchMedia` [API](https://developer.mozilla.org/en/DOM/window.matchMedia) provides a means of determining whether the current `document` matches a given [media query](https://developer.mozilla.org/En/CSS/Media_queries).  For example:

<pre class="language-javascript"><code class="language-javascript">if (window.matchMedia(' (min-width: 600px) ').matches) {  
    console.log('The viewport is at least 600 pixels wide');
} else { 
    console.log('The viewport is less than 600 pixels wide');
} 
</code></pre>

You can also use this API to add listeners that will be fired whenever the result of the media query changes.  In the above example the `matches` criteria will be met whenever the viewport is at least 600px wide.  If you wanted to receive notifications whenever the viewport crossed the 600px threshold you could use the following.

<pre class="language-javascript"><code class="language-javascript">var mediaQueryList = window.matchMedia(' (min-width: 600px) ');
mediaQueryList.addListener(function(mql) {
    if (mql.matches) {
        console.log('The viewport is at least 600 pixels wide');
    } else {
        console.log('The viewport is less than 600 pixels wide');
    }
});
</code></pre>

[If your browser supports window.matchMedia](http://caniuse.com/#feat=matchmedia) you can see this behavior live below by resizing your browser window under 600px on the following demo:

{% capture demo_height %}200{% endcapture %}
{% capture demo_path %}2012-06-15/matchMedia-example{% endcapture %}
{% capture demo_title %}machMedia example{% endcapture %}
{% include post/demo.html %}

Interestingly, it turns out you can also use this same technique to listen for the ```print``` media being applied when the user requests the document to be printed ([hat tip to Ben Wells](http://code.google.com/p/chromium/issues/detail?id=105743)):

<pre class="language-javascript"><code class="language-javascript">var mediaQueryList = window.matchMedia('print');
mediaQueryList.addListener(function(mql) {
    if (mql.matches) {
        console.log('onbeforeprint equivalent');
    } else {
        console.log('onafterprint equivalent');
    }
});
</code></pre>

This works great in Chrome 9+ and Safari 5.1 (with the exception of the fact that the [listeners fire twice in Chrome](http://code.google.com/p/chromium/issues/detail?id=105743)).  However, it doesn't work in Firefox or IE10, even though they both support ```window.matchMedia```.  

### Update (July 16th, 2012)

I created a bug on Firefox's issue tracker for this defect - [https://bugzilla.mozilla.org/show_bug.cgi?id=774398](https://bugzilla.mozilla.org/show_bug.cgi?id=774398).  I'll update this post when I hear back.

## Combining the Approaches

If you combine the two approaches you can detect print requests in IE 5+, Firefox 6+, Chrome 9+, and Safari 5.1+ (unfortunately Opera doesn't support either approach).

<pre class="language-javascript"><code class="language-javascript">(function() {
    var beforePrint = function() {
        console.log('Functionality to run before printing.');
    };
    var afterPrint = function() {
        console.log('Functionality to run after printing');
    };

    if (window.matchMedia) {
        var mediaQueryList = window.matchMedia('print');
        mediaQueryList.addListener(function(mql) {
            if (mql.matches) {
                beforePrint();
            } else {
                afterPrint();
            }
        });
    }

    window.onbeforeprint = beforePrint;
    window.onafterprint = afterPrint;
}());
</code></pre>

Note that your event handlers might potentially have to deal with the fact that they're going to be called twice per print request in Chrome.

## Why Would I Use This?

For most situations print stylesheets are all you need to prepare the document for printing.  But I can think of a couple practical uses of the JavaScript event.

### Responsive Print Images

One use is substituting a higher quality image for the purposes of printing.  Traditionally [web browsers have displayed images at 72dpi and most printers can handle 300dpi+](http://www.cssnewbie.com/print-friendly-images/).  While some newer devices are able to display images at much higher resolutions, most users are still using a screen that will show web images at much lower resolutions than their printer can handle.

Therefore an image that might look just fine on the user's screen might look fuzzy and grainy when printed out.  For most images this is acceptable, but it might be an issue for prominent images on regularly printed documents, like a company logo.  You probably want that to look crisp when printed out.

The [technique to work around this](http://www.alistapart.com/articles/hiresprinting) involves loading both images, showing only the lower quality one by default, then hiding the low quality image and showing the high quality one in the print stylesheet.  The main downfall of this approach is that the end user has to download both images regardless of whether they're going to print the page.  Users on 3G devices that have no intention or capability of printing the document will still have to download your high resolution logo.

With the ability to detect print requests in JavaScript you can substitute the higher quality image on the fly when the user requests the page to be printed.

<pre class="language-markup"><code class="language-markup">&lt;img src="low-quality.jpg" id="company_logo" alt="My Company" /&gt;

&lt;script&gt;
    (function() {
        var upgradeImage = function() {
            document.getElementById('company_logo')
                .setAttribute('src', 'high-quality.png'); 
        };

        if (window.matchMedia) {
            var mediaQueryList = window.matchMedia('print');
            mediaQueryList.addListener(upgradeImage);
        }

        window.onbeforeprint = upgradeImage;
    });
&lt;/script&gt;
</code></pre>

The nice thing about this approach is that users that never print will not have to download the high quality image.  This technique also degrades nicely; users with browsers that don't support the print events will simply print the lower quality image.

### Tracking Print Requests

Print events can also be used to track the number of times users print pages within a site or application.  Because of the lack of total browser support you wouldn't capture every print request, but this would be sufficient for getting a rough idea of how often people are printing.

<pre class="language-javascript line-numbers"><code class="language-javascript">(function() {
    var afterPrint = function() {
        // Here you would send an AJAX request to the server to track that a page
        // has been printed.  You could additionally pass the URL if you wanted to
        // track printing across an entire site or application.
    };

    if (window.matchMedia) {
        var mediaQueryList = window.matchMedia('print');
        mediaQueryList.addListener(function(mql) {
            if (!mql.matches) {
                afterPrint();
            }
        });
    }

    window.onafterprint = afterPrint;
}());</code></pre>

### So can I use this in a "real" application?

Sure, just make sure what you're doing degrades nicely for users using a browser in which the event will not be fired.

Can you think of any other practical uses of detecting print requests in JavaScript?  If so let me know in the comments.

### Update (July 16th, 2012)

Per the comments I've found that in addition to all the bugs mentioned above, certain browsers trigger the after print event early (with either `onafterprint` or the `window.matchMedia` handler implementation).

<pre class="language-markup line-numbers"><code class="language-markup">&lt;!DOCTYPE html&gt;
&lt;html&gt;
    &lt;head&gt;
        &lt;script&gt;
            var beforePrint = function() {
                document.getElementById('printImage').src = 
                    'http://stackoverflow.com/favicon.ico';
            };
            var afterPrint = function() {
                document.getElementById('printImage').src = 
                    'http://google.com/favicon.ico';
            };

            if (window.matchMedia) {
                var mediaQueryList = window.matchMedia('print');
                mediaQueryList.addListener(function(mql) {
                    if (mql.matches) {
                        beforePrint();
                    } else {
                        afterPrint();
                    }
                });
            }

            window.onbeforeprint = beforePrint;
            window.onafterprint = afterPrint;
        &lt;/script&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;img id="printImage" src="http://google.com/favicon.ico" /&gt;
    &lt;/body&gt;
&lt;/html&gt;</code></pre>

When printing the above document you would expect Stack Overflow's favicon to print, when in actuality Google's favicon prints.  Both events fire, but the after print event fires before the printing actually occurs, which in this case reverts the changes made in the before print event.

I was able to recreate this problem in Chrome and Firefox.

Therefore do not do anything that relies on the after print event to fix what the before print event did.  For responsive print images this shouldn't be an issue because there should be no harm leaving the higher quality image in place; the user has already downloaded it.
