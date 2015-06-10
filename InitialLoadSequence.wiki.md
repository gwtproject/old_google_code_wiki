# Design: Specifying an Initial Load Sequence for Programs with Code Splitting

## Introduction

To get good start-up performance from CodeSplitting, it helps to defer the loading of the leftovers fragment.  However, exclusive fragments cannot be used until the leftovers fragment has loaded.  Before the leftover fragment loads, the client can only successfully install "base" fragments, but a base fragment is more awkward to use.  Every base fragment has an assumed set of fragments that have previously been loaded, and that set of fragments must exactly match.  Thus, the theoretical ideal of just using base fragments would imply a combinatoric explosion in the set of base fragments that must be compiled.

For some applications, the problem is simpler, because the application writer knows exactly which split point will be reached first, and possibly which will be reached second and third, as well.  To support such users, GWT will allow specifying an initial load sequence.

## Goals

  * Allow specifying an initial load sequence for the calls to runAsync.
  * Take advantage of this sequence by compiling base fragments for split points in that sequence.

## Non-goals

  * Infer the initial load sequence automatically.  This has been briefly attempted, but the compiler's existing analyzer is not predicting good sequences even after the programs are massaged by hand.  The massaging was getting increasingly heavy handed.
  * Support multiple optimized load sequences.  That would be helpful but requires much more development work.

## Designation of Split Points

This is the first feature that required specifying a split point in a module file.  For simplicity, the first implementation will use JSNI references to the enclosing method.  This is unambiguous in practice, and can be made a compiler error whenever it's not.

To leave the door open for future development, the first character is required to be an '@' sign.  If a better way to designate split points is developed, we can phase it in gracefully by having the initial '@' sign can indicate the backwards-compatible JSNI reference.

## Designation of the Initial Load Sequence

The initial load sequence is designated in the application's GWT module file.  Since the sequence is literally a non-branching sequence of split points, a multi-valued configuration property can encode it.  If we later support optimizing for multiple initial load sequences, then this decision is worth revisiting.


## Acting on a Split Point Designation

ReplaceRunAsync now records the runAsync calls it replaced in a field in JProgram called runAsyncReplacements.

This field subsumes the human-readable "split point map", which is now computed on the fly in RecordSplitPoints.

CodeSplitter.pickInitialLoadSequence now runs in Precompile, and makes its decision based on the module configuration property and on JProgram.runAsyncReplacements.

The CodeSplitter then computes base fragments based on the assumed load order.  Additionally, it pokes a field in AsyncFragmentLoader to indicate the initial load sequence, so that AsyncFragmentLoader will know when to load a base fragment and when to load the leftovers fragment.