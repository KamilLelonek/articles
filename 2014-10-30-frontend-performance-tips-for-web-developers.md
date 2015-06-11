<p>
  <figure>
    <img src="/assets/images/frontend-performance/performance-fit.png" width="100%">
  </figure>
</p>

Optimization has always been a tough topic. [Donald Knuth](https://www.youtube.com/watch?v=75Ju0eM5T2c) said that *premature optimization is the root of all evil*. Performance is something that always causes emotions in web community. When we need it, we tune up our applications, try different and [new servers](http://www.rubyraptor.org/), use load balancers, speed up our applications by playing with threads and processes, but sometimes we forget about frontend at all. Server side is important too, how if we need to provide fast-responding websites, we have to optimize them in our browsers as well. **In this article I'd like to focus on client side and how to efficiently deliver content to end user.**

# What won't be about

Before I start, I'd like to precise what **won't** be mentioned in this article to give you a general overview about the overall content.

- Nginx setup and performance setting
- Caching
- Gzipping
- HTTP Headers (`cache-control`, `expires`, `vary`, `etag`, ...)
- CDN

**A lot of useful links covering those topics are included in resources under this blogpost.** Not a lot left, huh?

# Rationale

When I was preparing to this article I was thinking about something unique, what is not *mostly obvious*, especially for backend developers. We all know the server side quite well, but sometimes we don't have an opportunity to take care of client side - maybe because we have frontend developers in our team, maybe because we just don't want or like to do frontend at all, maybe because we only maintain server stuff and don't have any use cases on client side or for any other reason when we actually have some fear for touching a code that we are not experts in. A lot of us know how to configure workers, set proper HTTP headers, gzip and cache content and distribute it by CDN, but very few know about improving load processes for client side content, especially styles and scripts.

# So what is this about?

In this blogpost I'll focus on the following things:

**JS:**

- `defer`
- `async`

**CSS:**

- `dns-prefetch`
- `preconnect`
- `prefetch`
- `prerender`
- `subresource`

# JS

I'd like to focus on how we're loading scripts in our `html` pages. It's usually something like:

```
#!html
<!-- HTML4 and (x)HTML -->
<script type="text/javascript" src="javascript.js"></script>

<!-- HTML5 -->
<script src="javascript.js"></script>
```

It's so called *normal execution*, which is **the default behavior** that pauses HTML parsing until script is executed. It means that when we have heavy-logic scripts, displaying page content will be significantly delayed. Hence it would be so obvious to mention that **all scripts should be included in the bottom of `<body>`**, just above the closing tag.

## `defer`

```
#!html
<script defer src="javascript.js" onload="init()"></script>
```

Originally, this was nothing more than a hint to the browser that the script does not modify the DOM. Therefore the browser doesn't need to wait for the script to be evaluated, it can immediately go on parsing the HTML. On older systems this might save some parsing time.

However, IE has slightly changed the meaning of `defer`. **Any code inside deferred script tags is only executed when the page has been parsed entirely.**

`defer` downloads the file during HTML parsing and will only execute it after the parser has completed. **`defer` scripts are also guaranteed to be executed in the order they are declared in the document.** 

A positive effect of this attribute is that the **DOM will be available for your script**.

## `async`

```
#!html
<script async src="javascript.js" onload="init()"></script>
```

`async` downloads the file during HTML parsing and will pause the HTML parser to execute the script when it has finished downloading.

Don’t care when the script will be available? Asynchronous is the best of both worlds: **HTML parsing may continue and the script will be executed as soon as it’s ready.** I’d recommend this for scripts such as Google Analytics.

## When should I use what?

**Typically you want to use async where possible**, then defer, then no attribute. Here are some general rules to follow:

- If the script is **modular** and does **not rely on** any scripts then use async.
- If the script **relies on** or **is relied by** another script then use defer.
- If the script is **small** and is relied upon by an async script then use an inline script with no attributes placed above the async scripts.

Both `async` and `defer` scripts begin to download immediately without pausing the parser and both support an optional `onload` handler to indicate an initialization method contained in that script. 

The difference between `async` and `defer` centers around when the script is executed. Each `async` script executes at the first opportunity after it is finished downloading and before the window’s load event. This means it’s possible (and likely) that `async` scripts are not executed in the order in which they occur in the page. The `defer` scripts, on the other hand, are guaranteed to be executed in the order they occur in the page. That execution starts after parsing is completely finished, but before the document’s `DOMContentLoaded` event.

**The truth is that if you write your JavaScript effectively, you'll use the `async` attribute to 90% of your `script` elements.**

## Browser compatibility

> In older browsers that don't support `async` attribute, parser-inserted scripts block the parser; script-inserted scripts execute asynchronously in IE and WebKit, but synchronously in Opera and pre-4.0 Firefox. In Firefox 4.0, the async DOM property defaults to true for script-created scripts, so the default behavior matches the behavior of IE and WebKit. To request script-inserted external scripts be executed in the insertion order in browsers where the document.createElement("script").async evaluates to true (such as Firefox 4.0), set .async=false on the scripts you want to maintain order. Never call document.write() from an async script. In Gecko 1.9.2, calling document.write() has an unpredictable effect. In Gecko 2.0, calling document.write() from an async script has no effect (other than printing a warning to the error console).

| **Feature** | **Chrome** | **Firefox** | **IE** | **Opera** | **Safari** | **Android** | **IE mobile** |
|-------------|------------|-------------|--------|-----------|------------|-------------|---------------|
| `defer`     | Yes        | 3.6+        | 10+    | No        | Yes        | Yes         | No            |
| `async`     | Yes        | 3.5+        | 10+    | No        | Yes        | Yes         | No            |

**Documentation:**

- https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script
- http://www.quirksmode.org/js/placejs.html

# CSS

While loading JavaScripts in different ways is pretty *natural*, loading CSS asynchronously and improving load behavior, time and order only in HTML code may be surprisingly. I'd like to present some of attributes that are worth to be included in `<link>` tags to improve performance of you website.

## `dns-prefetch`

```
#!html
<link rel="dns-prefetch" href="https://api.twitter.com" />
```

`dns-prefetch` technique is very useful when your website links to related host names.

Browsers do not execute DNS lookups on those domains before visiting or starting fetching the content, so adding `rel="dns-prefetch"` indicates that particular hostnames should be resolved to their IP addresses.

This is a feature by which browser proactively performs domain name resolution on both links that the user may choose to follow as well as URLs for items referenced by the document including images, CSS, JavaScript, and so forth.

Prefetching is performed in the background, so that the DNS is likely to already have been resolved by the time the referenced items are actually needed.  This reduces latency when, for example, the user actually clicks a link.

## `preconnect`

`preconnect` is used to indicate origins from which resources will be fetched.

```
#!html
<link rel="preconnect" href="//www.example.com">
```

You can use `preconnect` to pre-open a new TCP connection so the browser can do this time intensive task while it waits for the rest of the HTML-page. By opening that TCP connection earlier we could save some valuable time.

> Initiating an early connection, which includes the DNS lookup, TCP handshake, and optional TLS negotiation, allows the user agent to mask the high costs of connection establishment latency.

## `prefetch`

```
#!html
<!-- full page -->
<link rel="prefetch" href="http://blog.arkency.com/2014/09/react-dot-js-and-google-charts/" />

<!-- just an image -->
<link rel="prefetch" href=http://blog.arkency.com/assets/images/react-js-google-charts/react-js-logo-fit.png" />
```

According to [w3.org](http://www.w3.org/TR/html5/links.html#link-type-prefetch), `rel=prefetch`:

> indicates that preemptively fetching and caching the specified resource is likely to be beneficial, as it is highly likely that the user will require this resource.
> There is no default type for resources given by the `prefetch` keyword.

Link prefetching is a browser mechanism, which utilizes browser idle time to download documents that the user *might* visit in the near future. `prefetch` hints have the lowest priority. The browser observes all of these hints and queues up each unique request to be prefetched when the browser is idle. There can be multiple hints per page, as it might make sense to prefetch multiple documents.

Keep in mind that prefetching does work across domains, including pulling cookies from those sites.

## `prerender`

```
#!html
<link rel="prerender" href="http://example.org/index.html">
```

According to [the documentation](https://developers.google.com/chrome/whitepapers/prerender):

Prerendering extends the concept of prefetching. Instead of just downloading the top-level resource, it does all of the work necessary to show the page to the user without actually showing it until the user clicks. Prerendering behaves similarly to if a user middle-clicked on a link on a page (opening it in a background tab) and then later switched to that tab. However, in prerendering, that *background tab* is totally hidden from the user, and when the user clicks, its contents are seamlessly swapped into the same tab the user was viewing. From the user’s perspective, the page simply loads much faster than before.

## `subresource`

```
#!html
<link rel=subresource href="critical/app.js">
<link rel=subresource href="critical/style.css">
```

`subresource` hint identifies critical resources required for current page load. These resources should be downloaded as early as possible, they are necessary for further page load so they must be fetched immediately.

## Browser support

- Using prefetching is possible in both Firefox and Chrome, and other browsers will probably implement it very soon.
- If you use prefetching on your website and the visitor browser do not support prefetching, nothing will happen. It is safe to use prefetching as browsers will either implement it, or completely ignore it.
- Google Chrome do not use the prefetch attribute and use prerender instead. This means that you have to implement both prefetch and prerender, as shown in the example below:

```
#!html
<link rel="prefetch"  href="http://www.example.com/"> <!-- Firefox -->
<link rel="prerender" href="http://www.example.com/"> <!-- Chrome  -->
```

**Documentation:**

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Controlling_DNS_prefetching
- http://w3c.github.io/resource-hints/
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Link_prefetching_FAQ
- https://www.igvita.com/2012/06/04/chrome-networking-dns-prefetch-and-tcp-preconnect/

# Bonus

## GA

GoogleAnalytics provides asynchronous syntax for loading tracking script.

The snippet below represents the minimum configuration needed to track a page asynchronously. It uses `_setAccount` to set the page's web property ID and then calls `_trackPageview` to send the tracking data back to the Google Analytics servers.

```
#!javascript
<script type="text/javascript">

  var _gaq = _gaq || [];
  _gaq.push(['_setAccount', 'UA-XXXXX-X']);
  _gaq.push(['_trackPageview']);

  (function() {
    var ga = document.createElement('script'); ga.type = 'text/javascript'; ga.async = true;
    ga.src = ('https:' == document.location.protocol ? 'https://ssl' : 'http://www') + '.google-analytics.com/ga.js';
    var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(ga, s);
  })();

</script>
```

You can read about it in the [official documentation](https://developers.google.com/analytics/devguides/collection/gajs/).

## PageSpeed or ySlow

There are some tools as extensions to our browsers for measuring load times and suggest some optimization improvements for them. Chrome offers [PageSpeed](https://chrome.google.com/webstore/detail/pagespeed-insights-by-goo/gplegfbjlmmehdoakndmohflojccocli), which I find a great tool and in Firefox you can use [ySlow](https://addons.mozilla.org/pl/firefox/addon/yslow/), which is quite nice web page performance analyzer.

# *TL;DR*

If you need just some short description you may check awesome slides http://bit.ly/preconnect-prefetch-prerender prepared by [Ilya Grigorik](https://www.igvita.com/), web performance guru at Google.

# Resources

- https://kinsta.com/learn/page-speed/
- https://medium.com/@luisvieira_gmr/html5-prefetch-1e54f6dda15d
- https://www.igvita.com/2014/01/31/optimizing-web-font-rendering-performance/
- http://betterexplained.com/articles/speed-up-your-javascript-load-time/
- http://www.mobify.com/blog/beginners-guide-to-http-cache-headers/
- https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Tips_for_authoring_fast-loading_HTML_pages
- http://www.yottaa.com/17-user-experience-and-site-performance-metrics-you-should-care-about-ebook
