_Team Engineering Meetings include only engineers (rather than the entire product team), so topics are limited to technical issues and discussion. Other topics, such as release schedules and so on, are covered in a separate meeting._

# Big Picture

**Are we interacting effectively with the GWT community on a technical level? Any adjustments?**
  * We need do a better job of posting meeting notes (i.e. actually need to it :-)
  * We need to make sure that individuals are sharing their ideas on GWTC early rather than waiting until a working implementation is finished
  * GWTC should always be the default location to have an exchange
  * Need to move forward on accepting some patches

**Are we being efficient in our GWTC discussions?**
  * Don't feel the need to reply to every post unless you really care and have something unique to valuable to add
  * We should emphasize use-case driven analysis of issues to avoid debating equivalent outcomes that hinge mostly on taste
  * We could clarify what an RR is and use them more effectively to maintain focus

**What is the procedure for accepting patches?**
  1. Find a patch you want to sponsor
  1. The sponsor iterates with patch author (publicly, on GWTC) until ready for code review
  1. Ensure patch author has a CLA on file
  1. Queue up a code review as with any GWT pending commit
  1. The commit comment should give credit to the patch author using the format specified here: http://subversion.tigris.org/hacking.html#crediting

**Do we need unanimity among committers for API changes?**
  * Yes. It's far worse to make an API change that ends up being wrong (because of backward compatibility) than to delay making an API change that ultimately does have the Right Design.
  * Note this will almost always be relevant only for API _additions_. To maintain backward compability, only _very_ rarely (if ever) would we remove APIs or change their behvaior.
  * If you truly don't know anything about an area of proposed change, you can always reply with "fhmp" (forever hold my peace), but please(!) don't use fhmp as a way to simpy avoid dealing with important changes.

**How many people should be included in a non-API code review?**
  * Should be proportional to the complexity of the change. One or two is typical.

# Status
  * Looking at a generalized way to measure "time to interactivity" at startup
  * Solid progress on optimized page bootstrapping
  * Progress on prototype drag and drop; will try to reconcile with recent DnD discussion/demo on GWTC
  * Collapsible panel in the works
  * Progress on prototype rich text editor; wiring in a spell checker is tricky but important to do even from the beginning
  * Progress on prototype auto-suggest
  * First iteration of image bundle almost finished and ready for review
    * next iteration will look at i18n
    * output as PNG supports all input formats best but IE problem with transparent PNG; may be able to work around using  CSS filter
    * better to wrap a SPAN around an IMG to gain clipping rather than using the funky CSS background-image trick
  * `GWTShellServlet` ought to set HTTP response headers correctly for certain URL patterns (e.g. ".cache.xxx"); great for demos and could serve to show what the preferred headers are
  * Refactoring of hosted mode interop to use a true Object as a native handle rather than an 'int'
    * this is the key to properly interacting with platform-specific GC mechanisms and  solving the Linux GC hosted mode crash
    * the new code moved more things to platform-independent code
    * IE back up and running; Mozilla C++ today; Safari in the next couple of days
  * Compiler refactoring of AST manipulation to keep it simpler, using Sandy's patch as a forcing function
    * Now able to pull in the peephole dead code elimination patch and some other code size reduction patches
    * It was noted that inlining methods in generated localized subclasses would be a nice size savings and doesn't require complex inlining

# Quick Consensus
  * JavaScriptObject API change needed for hosted mode refactoring. **Yes**
    * the ctor that used to take an `int` is being removed
    * breaks third-party subclasses ctors, but easy to fix and very likely low-impact
    * need to annouce to groups just to make sure it won't be a major problem
  * Don't we really need to fix the IE redundant image loading thing? **Yes**
  * Need to add "getAllMethods()" to JClassType? **Yes**
    * needed to support ImageBundleGenerator
    * most generators need the functionality to walk all the most-derived methods visible within a type
  * Could we add `getOutputDir()` to `GeneratorContext`? **No**
    * Needed for image bunde, but generally useful
    * Better to add `tryCreateOutputResource(partialPath)` that still encapsulates the ouput path and has semanics that allow transaction-like clean up if a generator fails
  * Should things like `ImageBundleGenerator` find resources in the claspath? **No**
    * Strongly recommend looking in module's source path instead
    * Implies adding `findResourceOnSourcePath()` to `GeneratorContext`
  * How fast should turnaround time on code reviews be?
    * Within 24 hours if it's a "I can't do it"
    * Within 48 hours if you are doing the review
    * Reviewee needs to be willing nag to keep the deadlines meaningful

