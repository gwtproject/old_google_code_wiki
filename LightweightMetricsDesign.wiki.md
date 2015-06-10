# Introduction

It is highly desirable to provide a "zero config" mechanism to gather coarse-grained runtime performance metrics to allow developers and production engineers to achieve increased visibility into how a GWT application behaves in the deployment environment.

## Goals
  * Provide data about the bootstrap process and RPC subsystem in code paths where user code cannot be added.
  * Support multiple modules on the same page.
  * Incur negligible overhead when no statistics collector function is defined.
  * Minimize observer effects by allowing for data aggregation to be performed in post-processing.

## Non-goals
  * Provide a general-purpose metrics system.
  * Support the "trivial" collector

# Specifics

The GWT module will look for a user-defined, global function and feed it event objects.  Event objects are simple JSON objects that have several "well-known" properties and additional, type-specific properties.
```
__gwtStatsEvent({
    evtGroup : 'phase1',
    millis : currentTimeInMillis,
    moduleName : 'com.foo.Module',
    subSystem : 'name',
    type : 'type',
    ...})
```

An Event Group (`evtGroup`) is analogous to a grouping of related events that can be assumed to follow a serial order. Each `type` can be interpreted as a checkpoint within an Event Group. The (`moduleName`, `subsystem`, `evtGroup`, `type`) tuple can be used as a "primary key" and should be used to correlate related event objects.

The `type` parameter may have the following values:
  * `end` : Indicates that a long-running process has finished
  * Any other value indicates a checkpoint in a long-running process. Note that a long-running process does **not** need an explicit begin 'type' since one can interpret the first received event of a given `evtGroup` as an implicit begin. However, each `evtGroup` should have an explicit `end type`.

No guarantee is made as to the latency with which an event object is delivered relative to when the event actually occurs.  Implementors of stats collectors should rely only on the `millis` property for timing information.

## Discussion points
  * The gwtStatsEvent / collector API is intended to express the incremental accumulation of properties about an (ongoing) event within the application.
  * The startup events only record timing information.
  * RPC records many different event objects for a single RPC request:
    * The `evtGroup` will be a monotonically-increasing number that can be used to correlate RPC checkpoint events
    * `end` event types will be generated, and several different `type` values will be used as checkpoints throughout the lifetime of an RPC request


# Future work
  * JS optimization to remove double-boolean-negation when an expression is going to be implicitly coerced to a boolean value.
  * It would be dubiously handy to have some Java syntactic sugar for building statically-evaluable JSO object literals without escaping into JSNI.
  * A GreaseMonkey or similar analysis tool to automatically compute timing deltas, make pretty timeline graphs, send results to a server, etc.

# GWT Startup Process
Implicit Event Object for a Startup event
```
{ 
  moduleName : <Module Name>,
  subSystem : 'startup',
  evtGroup : <Event Group>,
  millis : <Current Time In Millis>,
  type : <Event Type> 
}
```

![http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-startup.png](http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-startup.png)

# GWT RPC Event Diagram
Implicit Event Object for an RPC event
```
{ 
  moduleName: <Module Name>,
  subSystem: ‘rpc’,
  method: <Method Name>,
  evtGroup: <RPC counter>,
  millis: <Current Time In Millis>,
  type: <Event Type>
}
```

![http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-rpc.png](http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-rpc.png)

# GWT runAsync Event Descriptions
Implicit Event Object for a runAsync event
```
{ 
  moduleName: <Module Name>,
  subSystem: 'runAsync',
  evtGroup: <Event Group>,
  millis: <Current Time In Millis>,
  type: <Event Type>,
  fragment: <Fragment Number>,
  size: <Size of Uncompressed JS File>
}
```

The size is only present on end events, and currently is always -1.  In the
future, some linkers may supply the size of the downoladed code.  The fragment
number is the number of the deferred JavaScript file that is downloaded.  It
can be computed based on other information in the event stream, but it is
included for convenience.  The size and fragment number are only present at
all for download and leftoversDownload events.

![http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-runAsync.png](http://google-web-toolkit.googlecode.com/svn/wiki/LightweightMetricsDesign-runAsync.png)