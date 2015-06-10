# GWT Version 1.4 Development Plan
## Release Candidate Available

Download a copy [here](http://code.google.com/webtoolkit/download.html).

Details in the [GWT 1.4 RC blogpost](http://googlewebtoolkit.blogspot.com/2007/05/google-web-toolkit-14-release-candidate.html).

### Widgets/Library
  * RichTextArea ([#230](http://code.google.com/p/google-web-toolkit/issues/detail?id=230)) - jgw, knorton
    * done
  * DisclosurePanel (formerly CollapsiblePanel) - knorton
    * done
  * Widgets in Tabs - knorton
    * done
  * Splitter (horizontal and vertical) - knorton and jgw
    * done
  * SuggestBox - ecc ([#868](http://code.google.com/p/google-web-toolkit/issues/detail?id=868))
  * IncrementalCommand (formerly DeferredIterator)
    * done
  * ToggleButton,PushButton - ecc ([#869](http://code.google.com/p/google-web-toolkit/issues/detail?id=869))
    * done
  * DateTimeFormat ([#751](http://code.google.com/p/google-web-toolkit/issues/detail?id=751))
    * done
  * NumberFormat ([#752](http://code.google.com/p/google-web-toolkit/issues/detail?id=752))
    * done

### Development/Deployment
  * True JS output (litmus test: mash-up-able) ([#699](http://code.google.com/p/google-web-toolkit/issues/detail?id=699)) - _scottb_
    * in progress
  * Benchmarking subsystem integrated into JUnit - tobyr ([#702](http://code.google.com/p/google-web-toolkit/issues/detail?id=702))
    * done
  * Simplified external script inclusion (script ready-functions no longer necessary)
    * done
  * GWTShellServlet uses preferred HTTP headers to speed demos and educate for deployment
    * done
  * New JRE classes: ListIterator (TreeMap and SortedMap will _not_ be included in 1.4 after all) - hcc
    * done
  * Compiler reports file/line information on internal compiler errors (ICEs) - _scottb_
    * done
  * Automatic PNG transparency support in IE for ClippedImage - bruce
    * done
  * RemoteServiceServlet refactoring to be pluggable - mmendez
    * done
  * Hosted mode checks at JSNI boundaries ([#627](http://code.google.com/p/google-web-toolkit/issues/detail?id=627), [#736](http://code.google.com/p/google-web-toolkit/issues/detail?id=736)) - _jat_
    * done

### Performance
  * Compiler does simple inlining (e.g. localized Constants) - _scottb_
    * done
  * Compiler does additional low-hanging-fruit dead code elimination - _scottb_
    * done
  * Compiler does peephole JS output optimizations - _scottb_
    * done
  * Optimized bootstrap, with simplified host page inclusion mechanism (i.e. 

&lt;script src="module.js"&gt;

)
    * done
  * JRE size and speed optimizations: HashMap, ArrayList, Vector, HashMap, HashSet
    * done
  * ImageBundle, ImageBundleGenerator, clipping on !Image ([#753](http://code.google.com/p/google-web-toolkit/issues/detail?id=753), [#754](http://code.google.com/p/google-web-toolkit/issues/detail?id=754))
    * done

### Fixes
  * Hosted mode Linux crash ([#472](http://code.google.com/p/google-web-toolkit/issues/detail?id=472)) - _jat_
    * done
  * Hosted mode support for multiple modules on a page ([#484](http://code.google.com/p/google-web-toolkit/issues/detail?id=484)) - _jat_
    * done
  * Hosted mode JS use of toString on a wrapped Java object ([#775](http://code.google.com/p/google-web-toolkit/issues/detail?id=775), [#785](http://code.google.com/p/google-web-toolkit/issues/detail?id=785)) - _scottb_
    * done
  * Hosted mode memory leaks ([#846](http://code.google.com/p/google-web-toolkit/issues/detail?id=846)) - _jat_
  * GC-correct hosted mode in Safari - _jat_
    * done
  * Hosted mode caching bug (sometimes appears due to RPC after refresh) ([#711](http://code.google.com/p/google-web-toolkit/issues/detail?id=711))
  * Redundant image loading in IE ([#282](http://code.google.com/p/google-web-toolkit/issues/detail?id=282))
  * Trees in ScrollPanel (?)
  * HTMLTable bugs (already in code review -ecc)
  * ...Other bugs during a 3-4 day GWT bugfix...












