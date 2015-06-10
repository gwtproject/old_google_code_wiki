#How code is loaded on demand in the cross-site linker

# Introduction

The cross-site linker has two challenges for runAsync.  First, the code can't be loaded with XHR, because of the Some Origin Policy.  Second, the code cannot be loaded at the top level of any iframe, but instead must be inserted into an inner scope of a function.

This page describes the general strategy used.  The precise details, e.g. the label names, are subject to change.

# Inserting code into a function

In the initial code download, GWT code is wrapped in a function, and that function exports an "installCode" function as an expando off of the function itself:
```
function my_gwt_app() {
  var jslink = new Object();
  my_gwt_ap.installCode = function(code) {
    var install = new Function("jslink", code);
    install(jslink);
  }
  // rest of initial code
} 
```

Since the scope of each "new Function" call is different, all cross-island references must go through the jslink object.  New definitions must be installed like this:
```
my_gwt_app.installCode("jslink.foo = function foo() { }") 
```

Uses of an identifier from another island look like this:
```
my_gwt_app.installCode("jslink.foo();") 
```

The GWT compiler includes an optional pass that is controlled by the configuration parameter `compiler.predeclare.cross.island.references`.  It's false by default, but individual linkers can turn it on when needed.  Here's how the cross-site linker itself turns on this pass:
```
<set-property name="compiler.predeclare.cross.island.references" value="true">
  <when-linker-added name="xs" />
</set-property>
```


# Cross-Site code load

Cross-site code is loaded using class CrossLiteLoadingStrategy.

The general technique is like JSONP.  Script tags aren't subject to the Same Origin Policy, so they can be used to download the code.  The content downloaded via the script tag has been wrapped as follows:
```
__MODULE_FUNCTION__['runAsyncCallback3']('here is the downloaded code')
```

Before inserting the script tag, CrossSiteLoadingStrategy must insert a callback to handle that code.  The callback updates the runAsync book keeping and then uses the installCode function to actually load the code.

# Detecting download errors

Detecting download errors with XHR is straightforward, but trickier for script tags.  At the time of writing, the implementation uses all of the following attributes on the created script tag: onerror, onload, and onreadystatechange.  Additionally, onreadystatechange does not indicate whether the download actually succeeded or not.  Thus, the callbacks are all tolerant of bad code.  Correspondingly, LoadErrorHandler is now called LoadFinishedHandler.

# Retry

Retrying failed downloads is also tricky on some browsers.  There appears to be a heuristic in some browsers that if a script tag download for a URL fails three times, then no further downloads of that URL will be attempted.  To work around this, CrossSiteLoadingStrategy adds a serial number to each request.  Instead of requesting deferredjs/123.cache.js, it uses deferredjs/123.cache.js?serial=0 .