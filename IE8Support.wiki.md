# Introduction

IE8 was recently released, and while the existing GWT IE support mostly works, there are a few things that don't, and now seems like a good time to do some cleanup work as well.

There are a number of differences between IE8 and IE6/7, detailed below. The most directly affecting existing GWT libraries are:
  * CSS expressions removed (this is breaking PopupImplIE6)
  * Adds window.onhashchange event (this can be used for a much better history impl)
  * Adds data protocol for URIs (this can **finally** be used in resource bundles)

# Deferred Binding

While we could **two** user-agent targets (for a total of three: ie6, ie7, and ie8), this would have a significant impact on compile time, so we're going to leave "ie6" for both IE6 and IE7 support, and add "ie8" for IE8 and up.

We also need to ensure that you only get the IE8 deferred-binding if you are actually in IE8 standards-mode. Even when you're not in "compatibility view", if there's no DOCTYPE, you are essentially running IE7. MSDN encourages people to use the document.documentMode property to determine this, which we can do in the property provider.

If you want to force IE8-super-standards mode (even when the user has "compatibility mode" set), add the following meta tag:
```
<meta http-equiv="X-UA-Compatible" content="IE=8">
```

# Details
  * Modules that need to be updated:
    * UserAgent
    * dom.DOM
    * user.DOM
    * History
      * Update to use window.onhashchange (maybe make this standard, since it's in HTML5).
    * EmulationWithUserAgent
    * CoreWithUserAgent
    * ImageSrcIE6
    * ClippedImage
    * Focus
    * FormPanel
    * Popup
    * RichTextArea
    * TextBox
    * Tree

# IE8 Differences
  * input type='radio' name attribute can be changed dynamically
  * CSS expressions removed
  * Adds document.querySelectorAll()
  * Adds Cross-Domain Request, Cross-Domain Messaging
  * Adds native JSON object (ECMAScript 3.1 §15.12)
  * XHR
    * Adds XHR.timeout, XHR.ontimeout event
    * 6 connections per host
  * Ajax Navigation
    * Can modify the #hash fragment
    * Adds window.onhashchange event.
  * DOM Storage
    * localStorage object
  * Connectivity Status Indication
    * body online/offline events
  * Adds window.toStaticHTML
  * Adds data protocol for URIs
  * DOM prototypes now mutable
  * JS property getters/setters
  * No more weird IE zoom ratio issues (afaict)
  * CSS: position:fixed finally works (!)

# Open Questions
  * How can we force IE8 Standards Mode without requiring a meta tag?
    * Turns out "compatibility view" reports the user-agent as IE7, which will use the ie6 deferred-binding. This works fine.
    * IE8's "normal mode" without a DOCTYPE will give you essentially compatibility view. We need to detect this with (document.documentMode >= 8), and fall back to the ie6 deferred-binding.
  * Should the deferred-binding targets be IE6/7, IE8 -- or IE6, IE7/8?
    * Let's go with the former. IE6 and 7 are pretty close to one another, and IE7 is unlikely to ever change again in a significant way, whereas IE8 probably will.
  * Do we still need the ugly IE6/7 StyleInjectorImpl on IE8?
    * Looks like we do, and that it may need some tweaking (the result of document.createStyleSheet() is **not** an element).
  * See if the IE6/7 String.append() performance assumptions still apply to IE8
    * EmulationWithUserAgent module
    * TODO
  * See if we can get stack traces on IE8
    * CoreWithUserAgent module
    * Looks like we get the same useless error object we got on IE7.
  * Does the multiple fetch bug still exist on IE7/8?
    * ImageSrcIE6 module
    * Signs point to 'no'.
  * Do we still need to adjust scrollLeft for RTL mode?
    * Oh, for god's sake. It starts at 0 and goes **up** as you scroll left. Argh.

# (Possibly) Related Issues
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=1765
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=3236
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=3573
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=3588
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=3589
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=3590

# References
  * http://blogs.msdn.com/ie/archive/2009/03/12/site-compatibility-and-ie8.aspx
  * http://code.msdn.microsoft.com/ieteched08labs/Release/ProjectReleases.aspx?ReleaseId=1186
  * http://code.msdn.microsoft.com/ie8b2ajaxhol/Release/ProjectReleases.aspx?ReleaseId=1566
  * http://msdn.microsoft.com/en-us/library/cc288472(VS.85).aspx (MSDN IE8 page)
  * http://msdn.microsoft.com/en-us/library/cc304059(VS.85).aspx (Accessibility)
  * http://msdn.microsoft.com/en-us/library/dd282900(VS.85).aspx (DOM prototypes)
  * http://blogs.msdn.com/mikeormond/archive/2008/09/25/ie-8-compatibility-meta-tags-http-headers-user-agent-strings-etc-etc.aspx (Understanding Compatibility Mode)