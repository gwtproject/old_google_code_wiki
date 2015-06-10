_DomEventsAndMemoryLeaks is an older article with a bit more background on the underlying leak-prevention strategy._

There are two basic kinds of memory leaks:

  * Java-esque leaks: Objects that are unintentionally reachable, often in maps and lists.
  * IE-esque honest-to-god leaks: Objects that will never get collected because they've run afoul of a design flaw in IE (and some old versions of other browsers).

**Java-esque Leaks**

You can actually analyze the first case using standard Java tools in dev mode. They won't correctly tell you the **amount** of memory leaked, but they will tell you which objects are leaked, which is often quite illuminating. The actual amount of memory leaked in the browser should be proportional to the amount leaked in the JVM for "normal" objects; but for native/dom objects it can be quite different. Again, it's important to note that these aren't actually "leaks" in the C/malloc sense of the word -- they'll eventually get cleaned up, but could hang around longer than you want them to.

**IE Leaks**

Now we get to IE leaks (I refer to them as IE leaks because they're a very specific variety that only happen in IE, though to be fair they could occur in antediluvian versions of Mozilla). These are leaks that follow a very specific pattern (detailed in the references below) involving a circular reference between a Javascript object and a native COM object (DOM elements, XHR, XML elements, et al). To be very clear, no other circular ref will do -- it must involve one object of each type (again, see the references below for details).

As covered in detail on the GWT wiki (see below), the core GWT widget system has a very specific event-handling system that makes it impossible to trigger an IE leak, as long as you don't go straight to JSNI and hook up event handlers yourself (using things like Event.sinkEvents() is just fine). The same goes for built-ins like XHR/RequestBuilder, Timer, Window events, and the like.

**"Leaks" on Other Browsers**

Any remotely recent version of WebKit (Chrome, Safari, et al) or Gecko (Firefox) does not leak memory in the same way that IE does. They can, however, exhibit slightly different behavior when cleaning up circular references between native and Javascript objects. I can't speak to precisely what Firefox does, but Chrome's generational garbage collector won't collect objects involved in these references until it does a "major compaction". This happens less frequently than early-generation collections, so you may find native objects hanging around a bit longer than you'd expect.

**Observing Memory Usage**

First, let's start with the assumption that you're looking at memory in Task Manager (or Process Explorer). What you'll typically see is that, even in a reasonably-behaved web app, memory will increase to a point, then stabilize. If you're refreshing the app, you'll also typically see the memory appear to increase slowly over time -- but if you do it long enough, it will eventually drop back down, usually to a higher point than where it started, but stable. There's also a subtly confusing behavior when you minimize the browser window -- memory will drop to a very low level, then increase rapidly once you restore the window. The important thing to keep in mind here is that memory usage naturally fluctuates, and watching it increase over a single refresh is not sufficient to prove a leak. In a true leak situation, memory usage will increase without bound, and it takes some time to prove that.

**IE8 Caveat**

Observing memory usage on IE8 is a bit trickier. It appears to be allocating memory for a given instance of a page in a pool, which it then dumps on unload. This means that all of an app's memory will be freed when you close or refresh it. So if you want to find leaks on IE8, you need to watch its usage **while it's running**, which can take some time.

**Exacerbating Factors**

Ok, so why am I seeing gobs of memory wasted on IE? Often it's the case that your app isn't actually leaking memory on IE, but simply consuming a lot more than on other browsers. The most common cause I've run into has been DirectX filters, usually when trying to hack CSS opacity support. Whenever you see a CSS property that starts with "filter:", whether it be "filter:opacity(...)" or "filter:progid:DXImageTransform..." and so forth, you're using a DirectX filter. These little beasts can use a **lot** of memory, because they all seem to allocate an additional offscreen buffer for every element they're applied to.

My advice? Get rid of them. Accept that your app won't be as pretty in IE6/7 and move on. I once saw an app that blew ~1.5G of memory on IE7 (yes, that's gigabytes) that **wasn't leaking** -- it just had an enormous number of opacity filters. Getting rid of them dropped it down to a much more reasonable level.

**Things That Might Help**

  * Make sure you're not holding references to large widgets that are not visible (e.g., caching a reference to a popup once it's created so that you can avoid recreating it later -- this is a legitimate pattern, but can cost a non-trivial amount of memory).
  * Check your usage of native methods. Don't be paranoid about every native method, but do look for patterns like this:
```
native void hookSomething(JavaScriptObject something, Callback cb) /*-{
  something.onfoo = function() {
    cb.@my.Callback::onfoo();
  };
}-*/;
```

  * Check your usage of third-party libraries that wrap existing Javascript libraries. I've not had time to look into the details of a lot of these, but they necessarily include a lot of native code, which is likely to cause these kinds of problems.
  * Turn off all your IE opacity hacks (see above).
  * Use standard Java tools to make sure you don't have any Java-esque pseudo-leaks.

**Things That Almost Certainly Won't Help**

  * Calling IE's CollectGarbage() function. This will nudge the GC to run a bit early, but it already runs quite aggressively, and calling this function will definitely **not** collect anything that won't get collected normally.
  * Calling Widget.removeEventListener() or HandlerRegistration.removeHandler(). It is **never** necessary (or useful) to call these methods for any reason other than that you want to stop receiving events (this is one reason we tucked removeHandler() into HandlerRegistration -- most people never need to call it).
  * Nulling out references willy-nilly. Carefully cleaning up static references, especially clearing collections of references that you know you no longer need, can help mitigate regular Java pseudo-leaks, but will typically make no difference for browser-specific leaks.

IE Memory Leak References:
  * GWT: http://code.google.com/p/google-web-toolkit/wiki/DomEventsAndMemoryLeaks
  * Microsoft: http://support.microsoft.com/kb/830555
  * J15R: http://blog.j15r.com/2005/01/dhtml-leaks-like-sieve.html
  * PPK: http://www.quirksmode.org/blog/archives/2005/02/javascript_memo.html
  * IBM: http://www.ibm.com/developerworks/web/library/wa-memleak/

IE Memory Leak Tools:
  * Drip: http://blog.j15r.com/2005/05/drip-ie-leak-detector.html (later, http://outofhanwell.com/ieleak/index.php?title=Main_Page)
    * Do **not** use this tool. I wrote it over 5 years ago, it was modified by others, and appears to give false positives (and may be flat-out wrong) at this point.
  * Microsoft: http://blogs.msdn.com/b/gpde/archive/2009/08/03/javascript-memory-leak-detector-v2.aspx
    * Last time I checked, I believe this plugin dies on IE8, but it should work fine on IE7.
    * It has two modes: "potential leaks" and "real leaks". Make sure you're not looking at "potential", because it generates lots of false positives.