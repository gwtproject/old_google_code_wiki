# Directly-Evalable RPC

This project would change the implementation of GWT RPC to use payloads which can be passed directly to the JavaScript eval() function.


## Goals:

  * Preserve 90% compatibility with existing GWT RPC source
    * Retire CustomFieldSerializers with a supported replacement
  * Reduce size and complexity of client-side code
  * Expect to eliminate much of the client-side deserialization logic
    * Removal of type-identifying string literals from the RPC payload
    * Stop artificially rescuing fields that are referenced only by RPC serialization code
  * Ensure that payloads can be incrementally processed to avoid UI delays during deserialization
    * No recursive behavior
  * Provide per-permutation maps of Java field, method, and class names to (obfuscated) JavaScript names that can be consumed by third-parties
    * This opens up the possibility of making non-Java backends speak GWT RPC somewhat feasible instead of outright impossible
  * Stretch-goal: Provide prebuilt server-side class files or the ability to synthesize server-side classes to eliminate data- or reflection-driven dispatch

# Implementation

## Pinning constructors

GWT RPC serialization implicitly invokes the default constructor (unlike Java serialization).  Should we wish to continue this practice in order to preserve source compatibility, we may be able to avoid the need for SerializableTypeOracle to determine which classes are used if we have an explicit GWT.isReachable() or a syntactic equivalent:

```
package gwtimpl;
// Generated
class RPCSupport {
  @SuppressRescue
  public static Object constructForRPC(Class<?> c, JsArray fieldValues) {
    if (GWT.isReachable(Foo.class) || GWT.isReachable(FooSuperClass.class) || GWT.isReachable(FooImplementsThis.class)) {
      // This could just as easily be setting up a Map<Class, Builder>
      if (c == Foo.class) {
        // 80% case of default-serializable objects
        return setFieldsFoo(new Foo(), fieldValues);
      } else if (c == ClassWithCustomSerializer.class) {
        // 20% case of custom serializers
        return (new ClientCustomSerializer()).deserialize(constructForRPC(FlattenedFields.class, fieldValues));
      }
    }

   // ......

    throw new UnavailableInClientException(c);
  }

  // Assuming assignments to unread fields will be deleted and array lookups removed as no-ops
  // Ideally, if an object contains no rescued fields or only transient fields, this method will evaporate
  @SuppressRescue
  private native Foo setFieldsFoo(Foo o, JsArray fieldValues) /*-{
    o.@c.g.Foo::field1 = fieldValues[0];
    o.@c.g.Foo::field2 = fieldValues[1];
    return o;
  }-*/;
}
```

## Client-bound (web) payload format

The current RPC format is essentially a list of tokens, which requires client-side JavaScript code for dispatch.  In deRPC, the client-bound payload will consist of one or more JavaScript statements that, when interpreted via the eval() function, will result in exactly the correct JavaScript object graph that represents the payload.  Furthermore, the client-bound payload will be constructed so as to allow for incremental processing of the object graph.

```
(function payload(){
// Start with an interned string table
var stringTable = ['hello', 'world'];

// Back-references are stored in a common table
function group1(constructForRPCFunction, objectTable, payloadTable) {
  // constructForRPCFunction is a reference to the above constructForRPC method
  // The classRef is a reference to the class literal object for the given type
  objectTable[0] = constructForRPCFunction(classRef, [fieldValue, /*unused value*/, stringTable[q]]);
  objectTable[1] = constructForRPCFunction(classRef, [objectTable[0]]);
  return true;
}

// Additional groups as desired

function groupN(constructForRPCFunction, objectTable, payloadTable) {
  payloadTable[0] = constructForRPCFunction(classRef, [fieldValue, objectTable[1]]);
  return false;
}

// We replace the //EX and //OK prefixes with a boolean flag
return {shouldThrow = false, functions = [group1, group2, ..., groupN], other: metaData};
})();
```

The GWT module would receive the above payload and evaluate it in the lexical context of the module's permutation in the following manner:

```
class SerializationFunction extends JavaScriptObject {
  public final native boolean invoke(JavaScriptObject constructForRPCFunction, JsArray<JavaScriptObject> objectTable, JsArray<JavaScriptObject> payloadTable) /*-{
    return this(constructForRPCFunction, objectTable, payloadTable);
  }-*/;
}

class Payload extends JavaScriptObject {
    public native boolean shouldThrow() /*-{return this.shouldThrow;}-*/;
    public native JsArray<SerializationFunction> functions() /*-{return this.functions;}-*/;
    // Accessors for other metadata
}

class PayloadHandler {
  private static native JavaScriptObject getConstructForRPCReference() /*-{ ... }-*/;
  private static native Payload doEval(String payload) /*-{ ... }-*/;
  private static void doDeserialization(final Payload payload, final AsyncCallback<?> callback) {
    DeferredCommand.addCommand(new IncrementalCommand() {
      int index = 0;
      JsArray<JavaScriptObject> objectTable;
      JsArray<Object> payloadTable;

      public boolean execute() {
        boolean hasNext = payload.functions().get(index++).invoke(getConstructForRPCReference(), objectTable, payloadTable);
        if (!hasNext) {
          // We may have multiple-object payloads in the future
          Object payload = payloadTable.get(0);
          if (payload.shouldThrow()) {
            callback.onFailure((Throwable) payload);
          } else {
            callback.onSuccess(payload);
          }
        }
        return hasNext;
      }
    });
  }
}
```

If we are willing to change RPC the object-instantiation semantics to no longer call the default constructor on objects when they are deserialized and include final fields, then we could achieve a more efficient deserialization mechanism by slightly altering the JS prototype functions to accept an optional list of parameters which would be used to initialize the fields.

```
function SomeObject_0() {
  if (arguments.length) {
    this.field0 = arguments[0];
    this.field1 = arguments[1];
  }
}
```

Then the payload would contain:
```
objectTable[0] = new SomeObject_0(fieldValue, stringTable[q]]);
```

## Client-bound (hosted) payload format

In hosted mode, the data in the cookbook could be computed by looking up dispatch ids from the CompilingClassLoader, however this would not easily support -noserver style hosted-mode sessions (lacking a CompilingClassLoader), and the amount of data in the cookbook exceeds the amount of extra data that we would want to add to every payload.  Instead, we'll provide an RPC format flag to the server to indicate that string literals, rather than class literals, should be used as the first argument to the constructForRPCFunction.  We'll provide an option in RemoteServiceServlet to indicate whether or not hosted-mode connections should be allowed, which could be used to prevent class names from being used outside of a trusted IP space.

```
class RPCSupport {
  public static Object constructForRPCHostedMode(String typeName, JsArry fieldValues) {
    return constructForRPC(Class.forName(typeName), fieldValues);
  }
}

class RemoteServiceServlet {
  // Override to disallow hosted-mode payloads, which may leak information
  // about the class hierarchy
  protected boolean allowHostedModePayloads() {
    return true;
  }
}
```

## Server-bound payload format

Due to the mutable nature of client-side objects, serialization must be done synchronously.  The format will be very similar to the current server-bound payload, however, we can use GWT.isReachable() to cut down on the size of the dispatch table.  In web-mode, the payload will use the name of the object's constructor function and in hosted mode, the name of the type will be used.

# Custom serialization

The CustomFieldSerializer types will be removed in favor of annotating the DTOs with both a client- and server-specific instance of Serializer that will specify the contents of the fields of a given type.  If a client-side serializer is not specified, the "raw" field data will be used to construct the client-side object, and the fields of the client-side object will be directly visible to the server Serializer.  If both client- and server-Serializers are specified, the accumulated field data will be passed across the wire as a serialized FlattenedFields object to be passed into the peer Serializer.  It is not valid to specify a client-side Serializer without specifying a server-side Serializer because this case can be handled with a server-only Serializer that swizzles the type on the client.

|serverSerializer|clientSerializer|Effect|
|:---------------|:---------------|:-----|
|null            |null            |default serialization|
|class literal   |null            |default serialization of user-specified fields|
|null            |class literal   |compile-time error|
|class literal   |class literal   |FlattenedFields object serialized instead; always slower|

```
interface Accumulator<T> {
  // The field name is just used for lookups in the cookbook
  // Any set fields that have no entries in the cookbook are ignored
  void setField(String field, Serializable... o);
  void setField(String field, String... o);
  void setField(String field, int... i);
  ... other primitives ...

  // Used to specify an alternate implementation class in the client code
  // e.g. to support by-interface RPC
  // Possibly just set by type name to avoid server dependency on client code?
  void setSwizzleType(Class<?> type);
}

// This is a "flat" view of an object coming from the client or server
interface FlattenedFields {
  Serializable[] getFieldSerializable(String field);
  ... others to match Accumulator ...
}

// This can be used on both client and server, but server-only is more efficient
class Serializer<T extends Serializable> {
  public void serialize(Accumulator<T> a, T o);
  public T deserialize(FlattenedFields fields);
}
```

# Default serialization

Default serialization on the server is just a special case of custom serialization, where a generated DefaultSerializer is used that contains serialization logic for all Serializable types without custom serializers.  On the server, we could synthesize per-type implementations of Serializer to eliminate the need for reflection-based dispatch.

```
// A generated type
class DefaultSerializer implements Serializer<Serializable> {
  public void serialize(Accumulator<Serializable> acc, Serializable o) {
    if (o instanceof Foo) {
      serializeFoo(acc, o);
    } else if (o instanceof Bar) {
      .......
    }
  }

  public Serializable deserialize(FlattenedFields fields) {
    // This should never be called, since the default-serializable objects
    // are created directly from the payload
  }

  // Assume that reads to pruned fields result in 0, false, or null
  @SuppressRescue
  private static native void serializeFoo(Accumulator acc, Object o) /*-{
    acc.@c.g.g.Serializer::addField(...)("field1", o.@c.g.g.Foo::field1);
    acc.@c.g.g.Serializer::addField(...)("field2", o.@c.g.g.Foo::field2);
  }-*/;
}
```

# Changes to GWTCompiler

  * The compiler will export a per-permutation map of Java class names to the constructor functions, as well as field and method mapping, that will be used as a cookbook by server code when constructing RPC payloads.
    * Fields that are unused in the permutation will not have a mapping and will not be emitted in the payload.
    * This map must preserve the order of the fields that the JS constructor functions expect.  The expected order is defined to be the natural sort order of Java Strings.
    * The data will be exposed as a Linker artifact to allow for arbitrary format outputs and for resource-generating Linkers to remove unused data.
  * A GWT.getPermutationName() will be added to allow client-side code to determine the identity of the permutation so that the server-side code may know which permutation's cookbook to use
  * GWT.isReachable(Class<?> c) and @SuppressResucue will be added.
    * GWT.isReachable() is replaced by the compiler with a true or false literal and the top-half of the Java AST optimizations are run again
    * @SuppressRescue causes the Pruner's RescueVisitor to skip a Java method
      * Reads from pruned fields should result in 0, null, or false.
      * Assignments to pruned fields should be eliminated, although any side-effects should be preserved.