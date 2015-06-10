# GWT Protocol Buffers



## Goals
  * Feature-complete implementation of [proto2 protocol buffers](http://code.google.com/p/protobuf/), using a pay-for-what-you-use approach.
  * Ensure that protomessages can be exchanged client-to-server with minimal overhead.
  * Improve code maintainability over current implementations, since this project is positioned to be the canonical implementation of protocol buffers for GWT.
  * Provide both JSO-based web-mode code and jre-clean implementations to avoid the overhead of GWTTestCase and JSNI method dispatch cost in DevMode.
  * Develop in the open in order to maximize community feedback, recognizing google products as the primary customers.
    * Community feedback was incredibly helpful in driving quality and interoperability for RequestFactory and there's a potential opportunity cost not to use a similar process.

## Non-goals
  * Implement general pre-flight tooling hooks for GWTC (but they would be handy to have). Thus, changes to proto files will still require protocc to be run manually before a dev mode refresh

## Approach
The approach will be to port an existing, internal implementation over to the AutoBeans framework for core state management, serialization, and deserialization.  The code generator will be ported to run as a [protocc plugin](http://code.google.com/apis/protocolbuffers/docs/reference/cpp/google.protobuf.compiler.plugin.pb.html), in order to unify the tooling and compilation processes with other target languages since most developers using gwt-pb will necessarily use pbs on their server.

For example:
`protocc --python_out=myproject/python --plugin gwt-pb.sh .... project.proto`

## Engineering justifications
### Speed vs. code size
Optimize common paths, mainly gets/sets/has for speed.  As much as possible, we will try to make accessors for primitive properties and Strings read-through to underlying JSO. See note at the bottom of the page relating to support for 64-bit integer types.

Repeated messages (i.e. `List<FooMessage>`) will be lazily reified.  While this does add some overhead to tracking reification of any individual element, it's likely that a given client will not consume all fields within any given message, producing a net win.  Moreover, clients using incremental commands can amortize the cost of reification over several ticks of the event loop.

### Code size
Total API completeness, especially use of introspection APIs, works against code size.  GWT does not natively support reflection or introspection as that prevents effective dead-code elimination.  This requires baking additional data into the final JavaScript.  Additionally, the GWT compiler does not have good support for embedding structured data efficiently into the output, which further damages code size.

### Maintainability
By taking advantage of the pre-existing and actively maintained AutoBean system for generating the state-management and introspection code, improvements driven by gwt-pb will help RequestFactory users (including JSON-RPC consumers) and vice versa.

### Integration
Tying into protocc as a plugin will fit nicely with existing build processes.

Initially, the wire format will be based on an existing format for encoding protocol buffers for JavaScript clients.

## The plan
  * Land the lazy reification work to let AutoBeans use a Splittable as their backing store.
    * Using a Splittable makes them JSO-backed in web mode and JRE-backed in dev mode.
    * If a lightweight JSO library ever becomes public and stable, it won't be hard to switch from Splittable to that.
    * The deserialization of non-trivial values consists mainly of instantiating the Java type wrapper around the Splittable data.
  * Implement a fast copyFrom() or COW-style cloning mechanism for AutoBeans
    * Protobuffers are immutable, so it's very often the case that a new protobuffer is constructed as a copy of another.
    * On modern browsers, `JSON.parse(JSON.stringify(obj))` will likely be the fastest way to implement cloning.  A "manual" version of this may also be fastest for legacy browsers.
  * Allow AutoBeans to use the Builder pattern
    * Builder-style setters are already implemented; it's mainly a job of coupling the builder implementation to the (read-only) bean implemenation.
    * If the builder is accumulating state in a Splittable or other self-contained state object, the construction of the immutable bean is just a state hand-over, and could be as simple as cloning.
      * The Message.Builder only allows build() to be called once, so creating the returned Message looks like the deserialization case above, where the backing Splittable state object is handed over to the message.  Since all of the objects passed into the setters are "pre-reified", it's a low-cost operation.
      * Message.toBuilder() is basically a clone operation on the underlying state.
  * Adapt the current code generator to create AB-backed facade APIs to be invoked as a protocc plugin
    * A protocol plugin is just an executable that accepts a CodeGeneratorRequest protobuffer and emits a CodeGeneratorResponse protobuffer that contains the file contents to be emitted.  This will allow the generator to be run as a pre-GWTC-compile tool, while tying into the existing protocc invocation process.
    * I really like the composition-of-snippets approach taken by the existing code generator and would like to make that a standard tool available to GWT generator writers.
  * Find a way to minimize the code-size impact of the introspection APIs.
    * This might be implemented as compiler-aware immutable, lightweight datastructures to allow list or map values to be more efficiently encoded as JS literals.
    * There are many uses of java.lang.reflect.Type, many of which can be encoded as a `[ [ class literals ], [ number of params ] ]` the way AutoBeans currently embeds information for ParameterizationVisitors.

## Desired test methodology
  * In the same way we test the GWT JRE emulation types against a the real JRE types, the test framework for gwt-pb should have a mode where the tests are run against the "real" --java\_out implementation generated by the proto2 compiler.  By doing so, we can verify that the tests are themselves correct as well catch changes made to the proto2 APIs.
  * If possible, it would be useful to adapt existing tests of the --java\_out protocol buffers to run against the gwt-pb implementations to provide the converse of the above tests.
  * Create a benchmark app to be able to make objective comparisons between implementation choices.  Integrate this into a performance dashboard.

## Wire format

TBD.  Probably something JSON-esque since the encoding doesn't need to be human-friendly and the interpretation of the payload is entirely driven by the `.proto` definition.

```
{ 1 : { 55 : "Foo" }, 2 : "Bar", 3 : 42.1234 }
```

## Open questions

### Support for longs
The JavaScript language cannot represent a full 64-bit integer, since it has no true integral type. This means that long values of magnitude >= 2^51 cannot be encoded with a simple numeric representation  GWT provides an emulation layer for long values, but there's a definite performance hit when performing mathematical operations.  As such, we should discourage the use of 64-bit values for anything other than identifiers.

### Lightweight collections
The AutoBeans implementation types (and I suspect the pb impls as well) need list and map types, but generally don't require the full API provided by the java.util classes.  While it's out of scope for this project, having an implementation of LWRC's would definitely improve code size and efficiency.