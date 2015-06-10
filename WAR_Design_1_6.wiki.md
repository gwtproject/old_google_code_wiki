# Design Doc for 1.6 WAR structure

In 1.5, `GWTShellServlet` served resources directly off the classpath (for public files), or generated files from a temporary location.  This has the advantage of allowing fast refresh and resource updating, and making things "easy".  However, it has the downside of not leaving the user with something that's easy to deploy.

We will rectify the deployment issue in 1.6 by standardizing GWT around the "expanded WAR format".  The two **key principles** are:

  1. The result of running the GWT compiler (and possibly some associated tools/build rules) will be an expanded WAR directory structure that can be immediately deployed to a Java Servlet Container compatible web server.
  1. Hosted mode will operate using essentially the same format, in the same directory, to ensure that hosted and compiled web applications behave the same.

In 1.6, we always dump all resources directly into the WAR directory, which the server serves directly out of.  We automate in hosted mode what a build process would do.  This is triggered by the Hosted Browser actually executing a selection script; the selection script (when running hosted mode) forces a hosted mode link.  Subsequent `GWT.create()` calls may cause incremental links.

## Non-Goals / Off-the-table

  * Produce a JS library or anything other than a standard GWT application
  * Implement a more complex project structure based on the idea of a "source" war template which is automatically copied into the output war directory; external tools could do this.

## Proposed Simple GWT project structure
```
MyProject/
  build.xml                                      <ant build file with various targets to javac, gwtc, hosted, server, clean>
  src/
    <source code, GWT modules, public resources>
  war/
    MyProject.html                               <a host HTML page>
    MyProject.jsp                                <another host page>
    MyProject.css                                <non-GWT static resources>
    WEB-INF/
      web.xml                                    <contains servlet mappings>
      classes/
        <classes compiled by IDE or build rule>
      lib/
        gwt-servlet.jar
        <other server-side libs>
    qualified.ModuleName/
      qualified.ModuleName.nocache.js            <selection script>
      public_resource.gif
      generated_resource.png
      <other public and generated files>
      scripts/
        STRONGNAME1.cache.js                     <compiled script>
        STRONGNAME2.cache.js                     <compiled script>
        <other compiled scripts>
    qualified.ModuleName2/
      <as above>
  extra/
    qualified.ModuleName/
      <non-deployed linker artifacts>
    qualified.ModuleName2/
      <non-deployed linker artifacts>
```

Note: the `extra/` subfolder will not be produced by default.

## Changes in Suggested Best Practices

The most key user-facing change is that we no longer suggest that host HTML pages live in the public path of the application module.  Instead, we suggest that many static resources, most especially the host HTML page that includes a GWT module, should live in the `war` directory as a static resource rather than on the classpath.

## Command Line Options, old vs. new

In 1.5, the output level options work this way:

  * `-out` is respected by both GWTCompiler and GWTShell, it represents the "root" output folder
    * Defaults to the current working directory
  * Deployable files go into `out/qualified.moduleName/`
  * Non-deployed files go into `out/qualified.ModuleName-aux/`
  * Extra cruft is created in `out/.gwt-tmp/`

This situation is undesirable because it the out directory is not immediately useful without addition build rules and processing.  We propose for 1.6 that the output be in standard expanded WAR format.  We will retain `GWTCompiler` and `GWTShell` for backwards-compatibility with 1.5, but newly created projects will use new entry points, `Compile` and `HostedMode`.

  * `-war` is used to specify the WAR output folder
    * The default value is <current directory>/war
  * `-extra` is used to specify the output folder for non-deployed linker artifacts
    * The default value is to not produce extra output
  * `-server` can be used to specify what server should be run
    * The argument is the name of a class which implements a `ServletContainerLauncher` interface; details TBD
    * The default value is to launch a Jetty server, which will we distribute with GWT
    * `-noserver` can be used to run no server at all
  * `HostedMode` requires a list of modules to serve as the default argument
    * `-startupUrl` is optionally (but canonically) used to launch one or more urls in the hosted browser on startup, which is what GWTShell used to use the default argument for
  * `Compiler` can support a list of modules to compile, as opposed to just a single module
  * `-out` is no longer supported
  * `.gwt-tmp/` is no longer created in the output folder

## How Hosted Mode Will Work

We will no longer use `GWTShellServlet` and embedded Tomcat.  Instead, we will used Jetty by default, but allow other servers to be plugged in through a lightweight interface.

  1. `HostedMode` performs an initial link for each module specified on the command line
    1. A generated selection script is copied into `war/qualified.ModuleName/`
    1. The generated selection script (and other resources) will not override an existing compiled selection script; this is to prevent clobbering a compile
    1. No public or generator produced resources will be copied
  1. `HostedMode` starts the web server, targeting it at `war/`.
  1. `HostedMode` launches a hosted browser window for each `-startupUrl` specified on the command line
  1. The hosted browser requests the host HTML page from the server.
  1. The host HTML page loads the generated selection script for the included modules.
  1. The selection script (in hosted mode only) calls `window.external.initModule(<moduleName>)`
    1. Hosted mode triggers a new initial link, copying public resources and possibly updating the selection script
    1. If the selection script was updated, `initModule()` returns `true`, which signals the selection script to force a full-page refresh in order to load the updated selection script
  1. The selection script loads `hosted.html` into an `IFRAME`, and hosted mode continues bootstrapping as per 1.5
  1. Whenever new resources are generated from a `GWT.create()` resolution, an **incremental link** is performed
  1. If the user presses "Compile/Browse"
    1. The set of modules passed in on the command line are compiled
    1. Each module output directory, `war/qualified.ModuleName/` is cleaned out
    1. Each module is linked into its module output directory
    1. A web browser then loads the host HTML page, which loads the compiled selection script
    1. The compiled selection script detects that it is not running in the hosted browser, and therefore loads web mode normally

## Linker Stack Changes

The linker stack must now support hosted mode fully and correctly.  When a hosted browser begins to load a GWT application, a module-session is created, and an initial link is performed into the module-session directory.  This module-session is retained throughout the life of that hosted browser remaining on the page.  The initial link takes only public resources as inputs, but should produce a selection script based on zero `CompilationResult`s.  In the future, we may choose to synthesize a `HostedModeCompilationResult` to represent the actual deferred binding properties of the browser associated with the module-session.

As the application runs, calls to `GWT.create()` may generate new resources.  Each time new resources are generated, the linker stack will call a new `relink` method on all of the linkers in the stack, passing in the set of newly generated artifacts.  Each linker must retain its own internal state if it needs to consider previously-encountered artifacts.  The lifespan of a linker is guaranteed that each instance will be associated with exactly one module-session.

A further implication of the incremental resource generation is that the servlet container must also support loading resources that have not yet been created by the time the webapp context is instantiated.

## Coordination with Eclipse Plugin

Make sure that it expects to track the list of active modules for a given project.

## Deprecation Policy

The 1.6 release should deprecate the legacy entry points so that they may be removed entirely in the subsequent release.  Without a forcing function, there is no incentive for users to migrate to newer entry-points, leaving them as the proverbial albatross.

## Misc
  * JUnit will subclass GWTShell for now.  We will revisit this in 2.0 with OOPHM.

## Open Issues
  1. We could provide a "server reload" button on the main shell window in hosted mode.  Is this worth pursuing for 1.6?
  1. How do we isolate the web app classloader to ensure same behavior in deployment?
  1. `war/qualified.ModuleName/scripts/` is likely to change in 2.0 with runAsync
  1. Will this design work with Maven?

## Open Tasks
  1. Make sure app creator and samples follow this structure.
  1. We need to parse user's `web.xml` and verify that all module `<servlet>` tags have been declared; if not we should warn and provide the correct text to add.