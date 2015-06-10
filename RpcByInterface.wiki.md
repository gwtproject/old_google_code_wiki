# RPC-by-Interface

This document assumes the RPC custom serialization API defined in deRPC design doc, although the pattern could be adopted to the current CustomFieldSerializer implementation.

## Statement of problem

The current GWT RPC system is based around the serialization of concrete types.  In the general case, an identical concrete type must be available on both the client and the server.  While a custom Serializer can be used to change the actual type used in either the client or server, the serialized type must have knowledge of the Serializer to use.  This is problematic when RPC interfaces are declared using (abstract) types of which there are arbitrarily many implementations on the server.

```
interface MyService extends RemoteService {
  java.util.List<Object> someFunction();
}
```

Consider the case of java.util.List.  Besides the several concrete implementations in the java.util package, there are also inner classes used by the Arrays and Collections utility classes, plus any number of other synthetic implementations used by ORM implementations.  It's not currently feasible to provide Serializer annotations (or CustomFieldSerializers) for every one of these List implementations.

# Solution / API

The general solution is to allow arbitrary type hierarchies to be mapped onto an implementation of Serializer based on interface assignability in order to create an N:1 mapping of concrete types to a Serializer.  We will introduce the concept of a "Swizzler" that has the opportunity to choose an alternate concrete implementation.

We'll introduce the following additional annotations to the GWT RPC API:

`@ByInterface(Class<? extends Swizzler>[])` will be specified on the RemoteService declaration to indicate one or more server-side Swizzler types that will be used.  For any given type `T` being serialized by the server that is declared in the RemoteService interface to be accessed as type `I`, the Swizzler types will be examined to see if they implement `Swizzler<I>`.

```
// T is the base interface
// C is the concrete base type to be used in the client code
interface Swizzler<T, C extends T> {
  // An Accumulator is used to avoid the need to instantiate
  // an instance of the swizzled type on the server
  public void swizzle(Accumulator<C> acc, T o);
}
```

Because the declaration of Swizzler indicates the a new base type that all instances of the swizzled type will be replaced with, we are able to statically eliminate subtypes of the original interface that will never be used.  That is, in a module where only by-interface serialization is used, only one concrete implementation of List would necessarily be pinned by the RPC code.

The most specific `@ByInterface` match across all interfaces implemented by a RemoteService's will be used.  This would allow the following type to be added to the GWT RPC package as a drop-in decorator for RemoteService:

```
@ByInterface({LazyCollectionSwizzler.class, LazyListSwizzler.class, LazySetSwizzler.class, ....})
interface WithLazyCollections{}
```

# Solution / Implementation

By-interface Swizzlers will simply be added to the search mechanism used on the server when locating any by-concrete-type Serializers.  Swizzlers will have precedence since they stand to be able to reduce the number of concrete types in use in the client code.

The above example of LazyCollectionSwizzler might be written as follows:

```
public class LazyCollectionSwizzer implements Swizzler<Collection, LazyCollection> {
  public void swizzle(Accumulator<LazyCollection> acc, Collection o) {
    String rpcData;
    for (Object o : acc) {
      // ... make up a lazy RPC payload ...
    }

    acc.setField("data", rpcData);
  }
}
```

While the above example is somewhat fictional, it would be usable with Collection types that were from effectively foreign sources, such as ORM solutions (which may have synthesized the implementation at runtime).

It is perfectly acceptable to combine the Swizzler and Serializer interfaces for a type, and this would likely be the case if LazyCollectionSwizzler were actually implemented.