_NOTE: For the older, GWT 1.5 design please see [CompilationUnit\_1\_5](CompilationUnit_1_5.md)._

# Building `CompilationState` and `TypeOracle`
## Problem and Background
Before GWT 2.0, `CompilationState`, and `TypeOracle` were per-module singletons. Unfortunately, they were also currently quite stateful. In hosted mode in particular, types were added to `TypeOracle` and `CompilationState` over time as Generators ran, then those types were removed on refresh. This was not a problem as long as a user only has one "session" at a time open. However, in 2.0 out of process development mode encourages patterns where users might concurrently access the same module in multiple tabs, or multiple browsers. In this case, unpredictable behavior was likely to occur. This problem also manifested trying to do a JUnit test using multiple concurrent dev mode browsers.

For more pre-2.0 background, please see [CompilationUnit\_1\_5](CompilationUnit_1_5.md).

## Basic Solution
We create multiple instances of `CompilationState`, one per active dev mode sesions, or per concurrent compile. Each `CompilationState` state has its own `TypeOracle`, and neither of these can be reset or "refreshed". They operate in a forward-only fashion.. new types can be added via generators, but existing types cannot be removed or changed.

## Problem - Building `CompilationState` is too expensive
However, `CompilationState` is extremely expensive to build from scratch. One possible solution to the problem could have been to simply recycle old `CompilationState`s when a page is refreshed, using the old `CompilationState` refresh logic; if a user opened multiple tabs we'd simply have to spend tons of CPU creating multiple instances of `CompilationState`. However, this solution is not workable because generally speaking, we don't know when a page is being refreshed. In the current dev mode architecture, old sessions may hang around for several seconds after a new session is up and running, thwarting a recycling strategy. Something more sophisticated is needed.

## Refinement #1
In GWT 2.0, multiple independent `CompilationState` intances are all built from a shared singleton subsystem, `CompilationStateBuilder`. It makes sense for it to be a singleton, because ultimately all Java source files (henceforth, compilation units) that can possibly be managed, exist on the singleton system class path. `CompilationStateBuilder` is responsible for building new `CompilationState` instances using compiled compilation units that reflect the current contents of some subset of Java source files on the system class path. It also efficiently caches already-compiled units, and correctly and smartly recompiles stale units (and any units which depend on stale units). Ensuring cache correctness is a two-fold problem. First, `CompilationStateBuilder` must ensure that any time an underlying source file changes, the associated `CompilationUnit` is seen as stale. Secondly, `CompilationStateBuilder` must accurately track dependencies between units, such that one stale unit causes a cascading series of recompiles of units that transitively depend on the stale unit.

`CompilationStateBuilder` solves the first problem by mapping compiled units by `ContentId`. A `ContentId` consists of the main type name of a `CompilationUnit`, and a strong hash of the source file contents. This combination ensures that changes to source files can be identified in a lightweight manner. In addition, the `ContentId` instances themselves are cached by absolute resource location and last modified timestamp. As long as a resource's timstamp has not changed, the associated `ContentId` is assumed to be fresh.

`CompilationStateBuilder` solves the second problem using the existing dependency tracking mechanisms that have been in place since GWT 1.5 and earlier, and in addition it now also tracks JSNI references between units. These dependencies are recorded as `ContentId` instances, so that one `CompilationUnit`'s dependencies can be uniquely resolved to one particular revision of another `CompilationUnit`.

## Problem - `TypeOracle` is built from JDT data structures
Priior to GWT 2.0, JDT data structures were directly used to build `TypeOracle`. The JDT structures were transient and only existed until `TypeOracle` finished builder. If `TypeOracle` was later refreshed, old `TypeOracle` components would simply be reused. However, in the new design, multiple instances of a `TypeOracle` need to coexist, and new `TypeOracle` instances may be built from cache at a later time. Keeping the JDT structures cached in order to later build new `TypeOracle` data structures would consume an unacceptable amount of memory, and those JDT structures cannot natively be serialized to disk.  One possible solution could have been to modify `TypeOracle` so that individual components can be serialized, and a new `TypeOracle` could be built from deserialized pieces. However, the design of `TypeOracle` was not amenable to this approach.

## Refinement #2
Fortunately, John Tamplin already did the work necessary to build `TypeOracle` instances directly from bytecode as part of the Instant Hosted Mode effort. We took this one piece of IHM and thereby eliminated the need to cache JDT structures, instead we only cache bytecode.

## Problem - Generated Compilation Units
Prior to GWT 2.0, we cached the `CompilationUnit` for generated source files. If the same source file was regenerated with the exact same contents, the old `CompilationUnit` could be reused without recompiling. (This process was referred to as the GRAVEYARD in the existing wiki doc). We need to translate this idea to allow `CompilationStateBuilder` to also reuse generated units with the same source.

## Refinement #3
Generated units are simply cached by `ContentId` without regard to modification time. When a new source file is generated, a `ContentId` is created and an existing unit can simply be looked up in the cache.