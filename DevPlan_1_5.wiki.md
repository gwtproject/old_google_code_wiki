# GWT Version 1.5 Development Plan

_M2_ indicates fixes and enhancements to be completed in the second milestone release.
_RC_ indicates fixes and enhancements to be completed in the release candidate.

### Compiler
  * Java 1.5 Language Support ([#168](http://code.google.com/p/google-web-toolkit/issues/detail?id=168))
  * Method inlining across Java and Javascript methods ([#1111](http://code.google.com/p/google-web-toolkit/issues/detail?id=1111), [#1787](http://code.google.com/p/google-web-toolkit/issues/detail?id=1787))
  * String interning ([#1739](http://code.google.com/p/google-web-toolkit/issues/detail?id=1739))
  * Unused parameter removal and Redundant clinit suppression ([#1779](http://code.google.com/p/google-web-toolkit/issues/detail?id=1779))
  * Removal of empty switch cases ([#1177](http://code.google.com/p/google-web-toolkit/issues/detail?id=1177))

### JRE Emulation
  * Update JRE emulation classes for Java 1.5 language constructs
  * Update JRE emulation classes to implement java.io.Serializable for use with RPC ([#1139](http://code.google.com/p/google-web-toolkit/issues/detail?id=1139))
  * (_M2_) `LinkedList`, `TreeMap`, `EnumMap`, `EnumSet` ([#297](http://code.google.com/p/google-web-toolkit/issues/detail?id=297))
  * (_M2_) Add missing `String`/`Character` methods ([#490](http://code.google.com/p/google-web-toolkit/issues/detail?id=490), [#986](http://code.google.com/p/google-web-toolkit/issues/detail?id=986), [#1066](http://code.google.com/p/google-web-toolkit/issues/detail?id=1066), [#1729](http://code.google.com/p/google-web-toolkit/issues/detail?id=1729))
  * (_RC_) `LinkedHashMap`, `LinkedHashSet` ([#1489](http://code.google.com/p/google-web-toolkit/issues/detail?id=1489))

### Widgets/Libraries
  * Update UI libraries for Java 1.5 language constructs
  * Update RPC and TypeOracle for Java 1.5 language constructs
  * Add support for ARIA accessibility attributes ([#289](http://code.google.com/p/google-web-toolkit/issues/detail?id=289))
  * (_M2_) Add support for right-to-left layout and text direction ([#1401](http://code.google.com/p/google-web-toolkit/issues/detail?id=1401))
  * (_M2_) Optimizations to Widget library event sinking ([#908](http://code.google.com/p/google-web-toolkit/issues/detail?id=908), [#1491](http://code.google.com/p/google-web-toolkit/issues/detail?id=1491))
  * (_M2_) Add convenience methods to JavaScriptObject, Element, and Event (dependent upon 1.5 compiler optimizations to be affordable)
  * (_RC_) Add context menu events ([#24](http://code.google.com/p/google-web-toolkit/issues/detail?id=24))

### Fixes and More
  * (_M2_, partially done) Officially support strict/standard HTML doctype (various issues labeled [DocType](http://code.google.com/p/google-web-toolkit/issues/list?q=label:Category-DocType))
  * (_M2_) Replace the KitchenSink with a much prettier Showcase application, and add at least one good looking default style
  * (_RC_, partially done) All other issues labeled [Milestone:1\_5\_RC](http://code.google.com/p/google-web-toolkit/issues/list?q=label:Milestone-1_5_RC)