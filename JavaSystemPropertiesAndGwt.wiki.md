## Global
| **gwt.usearchives** | Turns on/off loading of GWTAR files (defaults to true) |
|:--------------------|:-------------------------------------------------------|
| **gwt.normalizeTimestamps** | Normalizes timestamps to 0 in generated archives (GWTAR or ZIP/JAR/WAR) |
| **gwt.persistentunitcache** | Turns on/off persistent unit cache (defaults to true)  |
| **gwt.persistentunitcachedir** | Directory where persistent unit cache will be generated (defaults to ../ relative to the `-war` dir) |
| **gwt.devjar**      | Overrides the computed GWT installation directory      |
| ~~gwt.nowarn.legacy.tools~~ | Suppress warning for using deprecated tools (there no longer are such tools) |

## Generators
| **gwt.resourceBundle.stripComments** | Turns off convenience comments in generated Java code for ClientBundles |
|:-------------------------------------|:------------------------------------------------------------------------|
| **gwt.imageResource.maxBundleSize**  | Max size of image (either height or width) in pixels above which it won't be bundled in a _strip image_ (defaults to 256) |

## Server side
| **com.google.gwt.safehtml.ForceCheckCompleteHtml** | Checks if the provided HTML string is complete in `SafeHtmlTemplates` on server side |
|:---------------------------------------------------|:-------------------------------------------------------------------------------------|
| **com.google.gwt.safehtml.ForceCheckValidUri**     | Checks if the provided URI is a valid Web Address (per RFC 3987bis) in `SafeUri` on server side. |
| **com.google.gwt.safecss.ForceCheckValidStyles**   | Checks for valid CSS property names and values in `SafeStyles` on server side.       |
| **gwt.codeserver.port**                            | Sets the port of SuperDevMode on the same server as a `RemoteServiceServlet`, to load serialization policies from. |
| **gwt.rf.ServiceLayerCache**                       | Turns on/off caching of `ServiceLayer` method return values (defaults to true)       |

## Performance
Most of those are for gathering metrics about GWT itself.
| **gwt.perflog** | Turns on performance logging |
|:----------------|:-----------------------------|
| **gwt.perfcounters** | Turns on performance counters logging |
| **gwt.memory.usage** | Turns on memory usage logging |
| **gwt.memory.dumpHeap** | Filename suffix to dump heaps into |
| **gwt.speedtracerlog** | Performance metrics output file name (enabled if set) |
| **gwt.speedtracerformat** | Speedtracer output format (raw or HTML, defaults to HTML) |
| **gwt.speedtracer.logProcessCpuTime** | Use cumulative multi-threaded process cpu time instead of wall time. Only one of logProcessCpuTime or logThreadCpuTime can be set, not both. |
| **gwt.speedtracer.logThreadCpuTime** | Use per thread cpu time instead of wall time. Only one of logProcessCpuTime or logThreadCpuTime can be set, not both. |
| **gwt.speedtracer.logGcTime** | Turn on logging summarizing gc time during an event |
| **gwt.speedtracer.logOverheadTime** | urn on logging estimating overhead used for speedtracer logging. |
| **gwt.speedtracer.disableJsniLogging** | Disable logging of JSNI calls and callbacks to reduce memory usage where the heap is already tight. |

## Compiler
| **gwt.jjs.javaArgs** | Parallel compiles: overrides the JVM args passed to a subprocess |
|:---------------------|:-----------------------------------------------------------------|
| **gwt.jjs.javaCommand** | Parallel compiles: overrides the command used to start a new JVM |
| **gwt.jjs.maxThreads** | Parallel compiles: the maximum number of in-process threads that will be used (defaults to 1) |
| **gwt.jjs.permutationWorkerFactory** | Parallel compiles: sets the list of PWF implementors (think of this as parallelization strategies) |
| **gwt.jjs.traceMethods** | Print debugging information about how the compiler optimizes (ClassName.Method:ClassName.Method...)|
| **gwt.jjs.stringInternerThreshold** | Minimum number of times a string must occur to be interned (defaults to 2) |
| **gwt.jjs.logFragmentMap** | Turns on fragment map logging (code splitting)                   |
| **gwt.coverage**     | Turns on coverage instrumentation in generated JavaScript. Must be set to a comma-separated list of source files to instrument, or a single file listing those source files one per line. Coverage data is stored in `localStorage` on `beforeunload` |
| **gwt.enableEnumOrdinalizerTracking** | Turns on logging for enum ordinalization                         |
| **gwt.jsinlinerMaxFnSize** | Maximum number of statements a function can have to be actually considered for inlining (defaults to 23) |
| **gwt.jsinlinerRatio** | Maximum allowable ratio of potential inlined complexity to initial complexity (defaults to 1.2) |

## DevMode
| **gwt.dashboard.notifierClass** | Name of a `DashboardNotifier` class to use to report events to a dashboard. Defaults to a no-op implementation. |
|:--------------------------------|:----------------------------------------------------------------------------------------------------------------|
| **gwt.nowarn.webapp.classpath** | Suppress warnings when servlet code access classes on the system classpath                                      |
| **gwt.dev.classDump**           | Tells the dev mode classloader to dump any rewritten bytecode to disk (debugging)                               |
| **gwt.dev.classDumpPath**       | Specifies where to write dumped classes (defaults to `rewritten-classes`)                                       |

## CheckForUpdates
| **gwt.debugLowLevelHttpGet** | Log debugging information  |
|:-----------------------------|:---------------------------|
| **gwt.forceVersionCheckNonNative** | Windows only: use the pure Java implementation |
| **gwt.forceVersionCheckURL** | Version check on a specific URL instead of the hardcoded one |
| **gwt.prefsPathName**        | Use a different user-prefs node to record version check info |

## JUnit
| **gwt.args** | Pass in a set of arguments to JUnitShell |
|:-------------|:-----------------------------------------|