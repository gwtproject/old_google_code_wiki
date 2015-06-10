# Status
Release Status 2/21 = 1 week from Wednesday
  * RichText: FF and IE looks good; unlikely on Safari; needs I18N
  * CollapsiblePanel done except for widgets, just needs review
  * Widgets in tabs is not a big deal, but not done; should do the same thing for CollapsiblePanel in terms of widget support
  * Splitter almost works; may be impossible to set vertical splitter height using anything other than pixels (which is bad b/c horizontal works just fine) Does this interact with mP's inline/computed style request?
    * What are the options? Not ship it? Ship it with a limitation? Keep trying to fix it for good?
    * Need to check in quirks vs. standard mode as well
    * Root problem is that you don't get an event when elements resize
    * Hard to make it work without adapting the size to be app-specific
    * Important use case is a splitter forcing its contained widget to scroll
  * SuggestBox done; can control popup by overriding; need to check that popping up on the bottom (left, right) of screen
  * DeferredIterator being finalized; Miguel will sponsor
  * DropDownToggleButton needed a bit of design generality; flickers on IE
  * Date/Number X Formatter/Parser but needs a bit of polish
  * Improved bootstrapping: hybrid mode weirdness (exposed by profiler and junit), need to verify that it can load from the filesystem
  * JS output: still doing a bit of research on namespace issue; UI guys need to at least look at dealing with UI event stuff
  * Benchmarking subsystem; Miguel will sponsor
  * Hosted mode refactoring: on track, IE does actually work
  * WidgetCollection class needs to be optimized (currently implemented the hard, slow way)
  * Rename 

&lt;script src="module.js"&gt;

 to 

&lt;script src="module.nocache.js"&gt;


  * ImageBundle needs to work with I18N
  * Benchmarking stuff has a problem on OS X due to SWT/AWT mixture
  * Tree in a ScrollPanel is better (but not perfect)
  * We need names on tasks for release
  * Bugfix days
    * No time before 2/21 -- we probably need to slip the schedule

# Quick Consensus

  * Should listener collections extend ArrayList instead of Vector? **Yes!!!**
  * LinkedHashMap not started; should we punt? **Yes**
  * What pattern should we use to I18Nify core components?
    * **Use our own I18N stuff**. It's very lightweight for the default case and not using it work be net confusing just to maintain a  "conceptual orthogonality" that may have no practical value
    * "Independently useful" (from our Design Axioms) doesn't **have** to mean separate modules, though generally it is still better to do so when feasible
    * Implication is to move I18N into RichText.gwt.xml (which is included in User.gwt.xml)

  * Okay to use AWT/Java2D as used in image bundle and benchmarking despite complications with SWT? **Yes**
    * The answer "You just can't use AWT in generators" is unacceptable.
    * The fact that integration is so tricky is very unfortunate, and we need to just think really hard about it.

  * Where do we put spell check driver and prefix trie?
    * **TBD** -- Emily will schedule a follow-up meeting

  * How do we proceed on vertical splitter?
    * The plan is to ship the horiz splitter, but unless we have a breakthrough, **vertical splitter will have to go into the incubator**...ouch

  * Do we need to create an experimental area in source control? **Yes** Where? **TBD**
    * It needs to be an "incubator" with real code reviews
    * Only for things that are intended to be part of the product but just can't quite be graduated yet










