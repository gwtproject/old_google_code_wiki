# Introduction
GWT Java API compatibility checker is a tool to check two versions of GWT for API source compatibility. In particular, the tool can ensure that newer releases of GWT are able to compile applications that previous versions could compile. A user can also specify explicit API changes that must be "white-listed," i.e., ignored by the tool. While we focus on GWT, this tool works for any Java API.

# Background
A key advantage of GWT is that client applications (developed using GWT) do not have to worry about supporting new browsers or newer versions of a previously supported browser. GWT handles such issues. For example, when IE8 will be released, client application developers will be able to easily support IE8 by simply compiling their applications against a newer version of GWT. In general, as GWT continues to improve both in the quality of the JavaScript code it generates and the number of platforms it supports, client application developers can tap into these improvements by simply compiling the Java versions of their applications against the latest version of GWT. For client application developers to be able to successfully tap into these improvements, new GWT releases must successfully compile applications that previous versions could compile. We call this property API source compatibility.


As GWT's popularity increases, the challenge of maintaining GWT's API source compatibility is going to increase due to two factors. First, the GWT code is likely to become more complex over time as the number of features and platforms it supports increases. Second, the number of contributors to GWT is likely to increase as well. With increasing number of contributors, the changes to the source code will increase in both magnitude and number. Therefore it is desirable to automate the task of checking API source compatibility.


Other evolving APIs, most notably the Java language API, also face a similar problem [[1](http://wiki.eclipse.org/index.php/Evolving_Java-based_APIs)]. Almost all of these APIs aim for binary compatibility, as specified in the Java language specification [[2](http://java.sun.com/docs/books/jls/third_edition/html/binaryComp.html)]. With binary compatibility, the aim for a new API release is to ensure that the client applications can use the class files of the new release in place of the class files of the previous release. There is no recompilation involved. Several tools exist for checking API binary compatibility [[3](http://clirr.sourceforge.net/), [4](http://sab39.netreach.com/Software/Japitools/Usage/japize/59/)]. To the best of our knowledge, no tool exists for checking API source compatibility.


Of course, there can be cases when breaking the API is necessary. To handle these cases, the tool should support "white-lists." Specifying a whitelist explicitly should force the developers to think twice before breaking an API. The whitelist would also help client applications in refactoring their code appropriately.

# Goals

The tool reports API differences between two collection of classes, as per API source compatibility. It supports all Java 1.5 features except annotations. It accepts a whitelist -- a list of the API changes that it must ignore. The changes it reports are:

  * Deleted API package
  * Deleted API classes and interfaces, including nested classes
  * Visibility changes (or more generally, changes in modifiers) of API classes that are common
  * Deleted API members
  * Visibility changes (or more generally, changes in modifiers) of API members that are common
  * Change in return type of API methods that are common
  * Change in exceptions of API methods that are common
  * Overloading of methods that prevent compiler from resolving method calls with a null parameter


Essentially, the tool should report an API change if any possible client application that could be compiled by the reference version of the API can no longer be compiled using the current version.


# Non-Goals

The tool currently does not check for binary compatibility though it could be easily extended to check for binary compatibility. Similarly, the tool does not handle java annotations. It could be easily extended to handle annotations.


# Use Cases

This section describes the use cases that ApiCompatibilityChecker will support.

  * Show command line help: When is run from the command line with no arguments, it outputs information explaining its usage.

```
java ApiCompatibilityChecker configFile

The ApiCompatibilityChecker tool requires a config file as an argument. The config file must specify two repositories of 
java source files: '_old' and '_new', which are to be compared for API source compatibility.

An optional whitelist is present at the end of the config file. The format of the whitelist is same as the output of the 
tool without the whitelist.

Each repository is specified by the following four properties:
name           specifies how the api should be refered to in the output
dirRoot        optional argument that specifies the base directory of all other file/directory names
sourceFiles    a colon-separated list of files/directories that specify the roots of the the filesystem trees to be included.
excludeFiles   a colon-separated lists of files/directories that specify the roots of the filesystem trees to be excluded

Example api.conf file:
name_old         gwtEmulator
dirRoot_old      ./
sourceFiles_old  dev/core/super:user/super:user/src
excludeFiles_old user/super/com/google/gwt/junit

name_new         gwtEmulatorCopy
dirRoot_new      ../gwt-14/
sourceFiles_new  dev/core:user/super:user/src
excludeFiles_new user/super/com/google/gwt/junit
```

  * Check compatibility: When ApiComptabilityChecker is provided a configuration file with information about the two repositories, it checks them for source compatibility and outputs the API difference. The API difference is a list of API change messages, with each API change message listed on a separate line. An API change message specifies two things: (i) a java package, class, method, or field in the existing API, (ii) the manner in which the API has changed. For example, an API change message could be "java.net MISSING" indicating that the package java.net has been removed. Each API change message is followed by an optional string containing additional information about the API change.

```
java ApiCompatibilityChecker configFile 
```

  * Check compatibility with expected failures whitelisted: The tool can handle a whitelist of API changes, specified in the "whitelist" section of the config file. These changes are excluded from the output.


We plan to put the tool into the GWT build, so that any update to the GWT trunk is checked for API compatibility against a reference version of GWT. Additionally, the tool will be used to check whether the JRE library support that GWT offers is compatible with the real JRE libraries. Ensuring this compatibility will allow developers to develop utility software that can be deployed on both the server-side and the client-side.


# Proposed Solution

We build on TypeOracle, a compilation framework provided by GWT. Checking API source compatibility involves the following major steps:

  1. Read the configuration file, and build TypeOracles for the two source trees.
  1. For each of the two source trees, identify the API packages, classes, methods, fields, and constructors. First the API packages are identified. Then the API classes within the packages are identified. Finally, the API members of each API class is identified. The API members include both the members defined in the class as well as those inherited by the class.
  1. Find the API differences. First the list of Api packages for the two sources is compared to obtain the list of intersecting API packages and the list of missing API packages. For each interesecting packages, the list of API classes in the package for the two sources is compared to obtain the list of intersecting API classes and the list of missing API classes. For each intersecting class, the list of API members (fields, methods, and constructors) in the class for the two sources is compared  to obtain the list of intersecting API members and the list of missing API members. Each intersecting API member and class is also checked for changes in modifiers (for example, if the keyword 'final' or 'abstract' was added) and changes in return types or the exceptions thrown.
  1. Remove duplicates from the API differences. We define duplicate as an API change message that is produced both for a class and its ancestor. For example, if 'java.lang.Object' had a public variable 'typeName' removed, we do not want the message to appear for all API classes.
  1. Remove API differences that are white-listed.


Identifying API elements is one of the key steps of this process. Below we describe our algorithm for identifying the API elements for different element types.

  * Package: A Java package is an API package if it contains at least one class or interface that is both public and outer.
  * Class/Interface: We categorize the API classes and interfaces into two categories: a) sub-classable API classes, b) non sub-classable API classes. Sub-classable API classes are API classes that are not 'final' and have a public or protected constructor. Now, we recursively define an API class as a Java class that meets the following criteria:
    * the package it is under is an API package.
    * if it is an outer class/interface, it is public.
    * if it is an inner class: Let the inner class be Inner and the enclosing class be Enclosing. Then Inner is an API class, if either of the two conditions are true: a) Inner is public and either Enclosing or one of the superclasses of enclosing is an API class.  b)  Inner is protected  and either Enclosing or one of the ancestor classes of Enclosing is an API class.
  * Member: A member (relative to an API class 'Enclosing') is an API member:
    * if it is defined or inherited by the class 'Enclosing'.
    * if it is either protected and Enclosing can be subclassed or it is public.

# Discussion

The ApiCompatibilityChecker tool uses TypeOracle to compute the API differences. TypeOracle in turn requires access to the java source code. Thus the tool can only work when Java source is available. An alternative to TypeOracle would be to use Java reflection which works on class files, and would not have required access to the source files. However, we chose TypeOracle over Java relfection because with TypeOracle, we could tap into the already existing GWT compiler infrastructure. We could not find an analogous tool (with compatible licenses) in case of Java Reflection. (Japize, a tool which uses Java reflection to check for binary compatibility, could have  been suitable, but the tool was under a GPL license.)  With Reflection, everything would have to be developed from scratch.

# References
  1. http://wiki.eclipse.org/index.php/Evolving_Java-based_APIs
  1. http://java.sun.com/docs/books/jls/third_edition/html/binaryComp.html
  1. http://clirr.sourceforge.net/
  1. http://sab39.netreach.com/Software/Japitools/Usage/japize/59/