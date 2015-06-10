## Introduction

GWT models pretty much everything as a resource. A resource might be a java source file, a class file, an html file, or a jar file. This design document describes how GWT determines which resources to include in a module.

## Glossary

### `ClassPathEntry`
A `ClassPathEntry` represents a single entry on the Java system classpath at startup.  These entries generally are either file system directory trees (`DirectoryClassPathEntry`) or jar/zip files (`ZipFileClassPathEntry`).

### `PathPrefix`
A `PathPrefix` provides the ability to select resources based on
their path names. For example, `PathPrefix("a/b")` only allows resource with path names that begin with a/b. A `PathPrefix` can select resources in a more general way by specifying a `ResourceFilter` and implementing its `allows` method. Finally, a `PathPrefix` also allows for rerooting of resources whereby the path of the resource is shortened.

### `PathPrefixSet`
A `PathPrefixSet` is a collection of `PathPrefix`. Note that if two or more `PathPrefix` with the same prefix but different resource filters are added to a `PathPrefixSet`, only the last one is kept.


### `Resource`
A `Resource` is any file that GWT uses.

## Determining the resources a GWT module includes
Conceptually, this determination is a three step process.

### Step 1: Construct the list of `ClassPathEntry` and the `PathPrefixSet` for a module
The resources included by a GWT module depends mainly on two things:
  1. `ClassPathEntry`: The order in which the `ClassPathEntry` are specified is important. This order is solely determined by the order in which entries are specified on the java classpath.
  1. `PathPrefixSet`: The order in which various `PathPrefix` were added to the `PathPrefixSet` is important. This order is determined by traversing the module inheritance tree rooted at the entry-point module in the lexical order. Plus, whether or not a `PathPrefix` causes its resources to be rerooted is important. Note that a `PathPrefix` with a source tag is not re-rooted but `PathPrefix` with public and super-src tags are re-rooted. For example, if the modules are as follows:
```
 Module MyApp.gwt.xml:
  <module>
    <inherits name="User.gwt.xml"/>
    <inherits name="Dev.gwt.xml"/>
    <source path="a/b" includes="*.java" />
    <entry-point class="..." />
  </module>

 Module User.gwt.xml: 
   <module>
     <source path="" />
   </module>

 Module Dev.gwt.xml: 
   <module>
     <super-source path="a" />
   </module>
```

> the `PathPrefixSet` constructed is as follows. Think of the modules being inlined to determine the lexical ordering of the `PathPrefix`.
```
  PathPrefixSet pps = new PathPrefixSet();
  p1 = new PathPrefix("");  // From module User.gwt.xml
  p2 = new PathPrefix("a/", WITH_REROOTING); // From module Dev.gwt.xml
  p3 = new PathPrefix("a/b/", filter allows only *.java);  // From module MyApp.gwt.xml

  pps.add(p1);
  pps.add(p2);
  pps.add(p3);
```

### Step 2: Determine the bag of resources for the list of `ClassPathEntry` and `PathPrefixSet`

For each `ClassPathEntry`, there is a deterministic collection of resources that is allowed by a `PathPrefixSet`. Specifically, each `Resource` matches a unique `PathPrefix` in a `PathPrefixSet`. The resource is allowed if the filter of the matching `PathPrefix` allows the resource. Lastly, the path of each allowed resource is computed. A resource is either a rerooted resource or a normal resource, depending on whether the `PathPrefix` that allows it requires rerooting or not. A rerooted resource's path is its path name following the `ClassPathEntry` and the `PathPrefix`. A normal resource's path is its name following the `ClassPathEntry`. In the example below, resources `r2` and `r5` match `PathPrefix` `p2`. Therefore, their (rerooted) path is "Test.java" and "a/b/Test.java" respectively.


```

      Initial path     Matching PathPrefix           allowed        Path
r1    Test.java               p1                       yes          Test.java
r2    a/Test.java             p2                       yes          Test.java (rerooted)
r3    a/b/Test.java           p3                       yes          a/b/Test.java
r4    a/b/gwt.gif             p3                       No           n/a
r5    a/a/b/Test.java         p2                       yes          a/b/Test.java (rerooted)
```


### Step 3: Determine which resource to include for each unique path
From this collection of resources, where each resource has an associated `PathPrefix`, `ClassPathEntry`, and a path, one resource is selected for each unique path. If there are multiple resources for a path, the following tie-breaker rules are used:
  1. A rerooted resource is preferred over a non-rerooted resource.
  1. A resource with a matching `PathPrefix` that was added to the `PathPrefixSet` later is preferred over a resource with a matching `PathPrefix` that was added earlier to the `PathPrefixSet`.
  1. A resource with earlier `ClassPathEntry` is preferred over a resource with a later `ClassPathEntry`.

These rules are ordered. Thus, a rerooted resource is always preferred over a non-rerooted resource, no matter how their matching `PathPrefix` and `ClassPathEntry` compare. To continue with the above example, the final resource map is:
```
Test.java => r2
a/b/Test.java => r5
```
because resource `r2` shadows `r1` and resource `r5` shadows `r3`. The examples above did not involve a tie-breaker rule involving `ClassPathEntry`, but it is easy to see how it would be used.

## Steps taken during a refresh
A `ResourceOracleImpl` refresh maintains the following invariants:
  1. If no resources change during the refresh,  the identities of all the Collections exposed by the `ResourceOracleImpl` remains the same.
  1. If any of the previous resource does not change during the refresh, its identity remains the same.

During a refresh, as a first step, the resource map is recomputed. If none of the resources in the map have changed, the previous collections are kept as is. This guarantees invariant 1. Furthermore, if a resource has not changed, the old resource is used instead of the new resource, thus guaranteeing invariant 2.

## When multiple `PathPrefix`es have the same path

**To be completed**

In the mean time, see https://gwt.googlesource.com/gwt/+/cfb6554a4bb21252f7b8d225fd78b2886874c3b6%5E!/
<br>This commit applies to GWT 2.1+