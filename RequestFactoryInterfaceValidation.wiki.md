Starting with GWT 2.4, RequestFactory interfaces must be validated before they can be used by the RequestFactory server code or JVM-based clients.  This document explains the mechanisms for validating those interfaces.



# Overview

**ERRATA This page incorrectly guides you to find requestfactory-apt.jar in the GWT\_TOOLS directory. Instead, Developers should find the requestfactory-apt.jar that came with their distribution. GWT contributors should use the appropriately dated jar in GWT\_TOOLS (at this writing requestfactory-apt-2011-08-18.jar), or build their own via `ant requestfactory`. Note also that the screen shots are aimed at GWT contributors.**

The RequestFactory annotation processor will validate the RequestFactory interface declarations and ensure that the mapping of proxy properties and context methods to their domain types is valid.  The manner in which the errors are reported depends on the method by which the annotation processor is invoked.

In addition to validating the interfaces, the annotation processor also generates additional Java types which embed pre-computed metadata that is required by the RequestFactory server components.  Users of `RequestFactorySource` must also run the annotation processor in order to provide the client code with obfuscated type token mappings.  In the client-only case, the server domain types are not required.

If the validation process is not completed, a runtime error message will be generated:
> The RequestFactory ValidationTool must be run for the com.example.shared.MyRequestFactory RequestFactory type

# Annotation Processor

It most convenient for both the shared RequestFactory interfaces and their server domain counterparts to be available on the sourcepath or classpath during the compilation process.  If this is not practical, see the `ValidationTool` section for information on post-processing pre-compiled shared interfaces and domain types as part of a deployment process.

## javac builds

Users using javac to compile their entire project at once need only to include the `requestfactory-apt.jar` on the build classpath.  The compiler will automatically load the annotation processor from the JAR file.  The shared proxy interfaces must be included in the list of java files being compiled.

## IDE configuration

### Eclipse
Open the project properties dialog and navigate to `Java Compiler -> Annotation Processing`.  Ensure that annotation processing is enabled -- by default it is not.

<img src='http://i.imgur.com/7upSN.png' />

Add the `requestfactory-apt.jar` to the factory path.

<img src='http://i.imgur.com/lMw0T.png' />

The project will need to be rebuilt, but once this is done, any RequestFactory validation errors or warnings will be displayed in the editor upon saving:

<img src='http://i.imgur.com/QQGHU.png' />

# ValidationTool

Not all build processes are amenable to compiling the shared proxy interfaces and the domain types at the same time.  Cases where the `ProxyForName` and `ServiceName` annotation are used are typical of this build configuration.  In these cases, it is possible to post-process pre-compiled shared and domain types using the RequestFactory `ValidationTool`.

To run the `ValidationTool` it is necessary to have the shared interfaces and domain types available on the classpath.  The binary names of the RequestFactory interfaces are provided to the tool as well as an output location, which may be a directory or a jar file.  The tool will produce one or more class files that contain precomputed metadata that will be used by the RequestFactory server code.
```
#!/bin/sh
java \
  -cp requestfactory-apt.jar:requestfactory-server.jar:your-shared-classes.jar:your-server-classes.jar \
  com.google.web.bindery.requestfactory.apt.ValidationTool \
  /path/to/output.jar \
  com.example.shared.MyRequestFactory \
  com.example.shared.AnotherRequestFactory
```

An optional client-only mode will produce the metadata required by `RequestFactorySource` clients using only the shared interfaces.  This mode is activated by adding a `-client` flag before the output location.

## Maven builds
Use the following recipe to run the ValidationTool. If you are using Maven with Google Plugin for Eclipse (see WorkingWithMaven), you do not need to configure Eclipse with the requestfactory-apt.jar as described above.

```
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.5.1</version>
        <configuration>
          <source>1.6</source>
          <target>1.6</target>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>com.google.web.bindery</groupId>
            <artifactId>requestfactory-apt</artifactId>
            <version>${gwtVersion}</version>
          </dependency>
        </dependencies>
      </plugin>
```

In order for your Eclipse project to be automatically configured from the POM, you'll need to install the [m2e-apt connector](https://community.jboss.org/en/tools/blog/2012/05/20/annotation-processing-support-in-m2e-or-m2e-apt-100-is-out). To install it, go to Help > Install New Software... and use `http://download.jboss.org/jbosstools/updates/m2e-extensions/m2e-apt` as the update site.