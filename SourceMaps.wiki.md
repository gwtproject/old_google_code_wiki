# Introduction

This document explains the addition of Closure-style [Source Maps](http://code.google.com/p/closure-compiler/wiki/SourceMaps) to Google Web Toolkit for stack-trace deobfuscation.

# Details

In GWT, when an exception is thrown, there are three possibilities:

  1. You are running in DevMode and have perfect fidelity stack traces from the JVM
  1. You are running with compiler.stackMode = emulated and get perfect fidelity stack traces
  1. You are running with native browser stack traces and get method resolution only.

In the first two cases, you can get exact stack traces when an exception is thrown, exact meaning that the actual line number of the offending code is shown and the original file it resides in. However, DevMode and emulated stack traces are not used in production code.

GWT's current native browser stack trace support is not exact for several reasons:

  1. It only is able to use method names along with `SymbolMapsLinker` to find the start of methods, it does not show you where the error occured **within a method**.
  1. Compiler optimizations like inlining strip away stack frames so that the original enclosing method is lost.
  1. Anonymous functions have no symbol to lookup in symbol maps.

# Source Maps

Unlike GWT's current Symbol Map support, SourceMaps not only record a mapping from Java identifier to obfuscated Javascript identifier, SourceMaps record a complete mapping of Javascript source ranges to Java source ranges. Every (line, column) - (ending line, ending column) span within the compiled script has a unique mapping to a Java file name and line number. Thus, if the browser could tell us, for each method on the stack, the exact column and line number, we could perfectly map this to the original Java source.

# Enter Chrome / V8

Unfortunately, most browsers don't supply line and column information. In fact, most don't even supply line numbers.  The Chrome browser features an Error.stack property on stack traces which not only includes a full Javascript stack trace, but provides column number and line number information within the script for each frame on the stack. Firefox is promising full support for this soon (https://wiki.mozilla.org/DevTools/Features/SourceMap)

There's just one catch: GWT code fragments are loaded in many different ways. Some are loaded as src URLs using 

&lt;iframe&gt;

 or 

&lt;script&gt;

 tags, and some are loaded by using XHR and the equivalent of JS eval(). However, whenever a script comes from an eval() or via dynamic script.text injection, it loses origination information, and since it doesn't have a filename to associate with itself, it also stops yielding line, column information.

The workaround is to use a **magic** comment in the JS that allows overriding of the source location. http://blog.getfirebug.com/2009/08/11/give-your-eval-a-name-with-sourceurl/

# Compiler Changes

SourceMap support leverages the existing SOYC information recorded in the compiler. The SOYC information is converted to an artifact and added to the artifacts to be linked. The `SymbolMapsLinker` picks this up and writes out the symbol maps, after adjusting them for any prefix or suffix code added by the primary linkers.

For each permutation, for each runAsync fragment, a sourcemap file is written out with the filename `<strongName>_sourceMap<fragmentNumber>.json`. Each fragment written to disk is modified to contain a `//@ sourceUrl=<fragmentNumber>.js` annotation as the last line.

# Runtime Changes

`StackTraceCreator` has been modified for Chrome to analyze the `Error.stack` property of an exception, and encode column and line number information in each StackTraceElement. Since the Java StackTraceElement class only includes a field for line numbers, the line and column information are encoded in a single 32-bit number.

On the server, `StackTraceDeobfuscator` is modified to first try regular symbol maps as a fallback. Then, using the stack trace from Chrome in the form `filename.js:linenumber`, it looks for the appropriate source map (if filename is a number, or initial fragment), and refines the `StackTraceElement` information based on information from the SourceMap.

# Turning on SourceMaps

To enable source maps (Chrome only), simply add



&lt;set-property name="compiler.useSourceMaps" value="true"/&gt;



to one of your module files.

Keep in mind, this will yield larger jar/war files as the source maps take up more space.