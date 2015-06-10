# Introduction

GWT builds are highly parallelizable.  The expensive part of the compilation is the part that optimizes and generates code for each browser-specific permutation.  By compiling the permutations in parallel, ideally on separate machines, overall build time can be greatly reduced.


# Details

A distributed GWT build has three stages: Precompile, CompilePerms, and Link.  Precompile and Link normally run on the main build node.  One instance of CompilePerms runs per permutation, and those instances can run in parallel, on multiple nodes.

All three commands take a -workDir argument.  Specify a directory where they can place their outputs and read their inputs.  The items placed there by Precompile need to be shared between the three stages -- either via a shared filesystem or by copying the contents after Precompile to the machines that run CompilePerms, then copying back all the files that result from all the CompilePerms runs before running Link.

The Precompile command takes a module name as input and emits a file precompilation.ser into the work directory.  Here's a typical command line:
```
  java -cp ... com.google.gwt.dev.Precompile com.google.gwt.sample.hello.Hello \
    -workDir work
```
The CompilePerms command takes a module name and a list of permutations that it should compile.  The permutations are numbered sequentially starting from 0.  It is up to the embedding build system to know the number of permutations (which can be obtained from the file `permCount.txt` in the workDir after running Precompile) and run the right number of instances of CompilePerms.  Here is a typical set of commands to run:
```
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 0
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 1
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 2
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 3
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 4
  java -cp ... com.google.gwt.dev.CompilePerms com.google.gwt.sample.hello.Hello \
    -workDir work
    -perms 5
```
Link combines all the permutation results into a final output.  It takes -war and -extra arguments to indicate where the public and private output should go.
```
  java -cp ... com.google.gwt.dev.Link com.google.gwt.sample.hello.Hello \
    -workDir work
    -war www
    -extra aux
```

# Classpath

All inputs to these commands come from the Java classpath, just as in a one-stage compile using the Compiler entry point.  The Java classpath of all three commands should include: the Java source code of your application, gwt-dev.jar, gwt-user.jar, any compiler-visible resources such as ui.xml and .png files, and the compiled versions of any generators and linkers you have added to your build.

# Parallel build systems

The three-stage entry points described above allow GWT to be used in distributed builds, but GWT does not include a distributed builder.  Any project wanting a distributed build needs to work out a distributed build system.  It's possible to get by using ssh and scp, but it's probably easier to use a dedicated tool like Hudson, CruiseControl, or Pulse.