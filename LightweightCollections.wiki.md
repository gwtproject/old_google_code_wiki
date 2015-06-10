# Lightweight Collections in GWT

We've found that the API semantics for the JRE collections classes are not an ideal match for the constraints of running inside browsers, especially mobile browsers, for several reasons.

## Background and Motivation
_The JRE collections framework encode assumptions about the relationship between types, objects, and their implementation making them highly coupled, even at the top of the hierarchy._ For example, the `Map` interface has methods such as `values()`, `entrySet()`, `keySet()` that make very specific guarantees about how the returned objects interact with the map from which they derive. These sorts of prescribed dependencies between collection types (e.g. using a `Map` implies the use of `Set` and `Collection`) and the behavior of collection objects themselves prevent simple implementations. Repeated attempts to optimize GWT's emulated JRE collections show a relatively high code-size cost for even minimal usage of JRE collections because, for example, using a `Map` implies using a `Set` and a `Collection`.

_Too few collection types forces runtime enforcement of behaviors that could be more efficiently modeled statically._ A common example is a class `T` that wishes to return a reference an internal list. However, the `List` interface has mutators (e.g. `add()`) which `T` would not want to make possible "from the outside." The traditional best practice of calling `Collections.unmodifiableList()` on the returned list reference requires the allocation of an extra object (which generates extra GC runs that can turn pathological in IE) as well extra code that must be generated simply to implement `throw new UnsupportedOperationException()` when a mutator is called on the wrapper object. If instead there were an `ImmutableList` type that the list collections implement, class `T` could simply have returned an immutable reference, which would ensure attempts to modify the list are caught at compile-time (i.e. because `ImmutableList` has no mutators) and which would prevent extra object allocations and potentially even facilitate inlining at compile time.

_Under-specified semantics in JRE collections lead to defensive programming in application code._ Java developers are encouraged to use the least specific type that has the desired semantics. For example, such advice would say use `List` in your code rather than `ArrayList`. However, there is no guarantee that `List#get(int i)` returns in constant time, so a generic algorithm written in terms of `List` cannot even be assured of its expected time complexity. The JRE collections includes the tag interface `RandomAccess` as a way to workaround the ambiguity, but forcing a check such as `myList instanceof RandomAccess` is time consuming at runtime (where time is precious) and, worse, can defeat the GWT compiler's optimizer.

## Goals
  * Absolute minimum size of compiled script. It should be impossible to write more succinct JavaScript by hand.
  * Absolute maximum speed. It should be impossible to write faster JavaScript by hand.
  * Explicit and well-publicized guarantees of time complexity for each operation. By removing fuzziness, developers can feel confident that they need not be defensive (e.g. by cloning collections unnecessarily).
  * The vocabulary of types should be rich enough to avoid the temptation of creating wrapper types. In other words, a method such as `unmodifiableList()` should never even seem to be necessary.
  * Construction and mutation semantics should minimize object creation. Ideally, the collections themselves could enforce patterns such as having only a singleton instance of an empty list.
  * The same set of collection types must work on the client and the server. We must avoid burdening application programmers with the asymmetry of using JRE collections on the server and GWT collections on the client.
  * Collections must be efficiently serializable over GWT RPC.
  * Collections must be able to be `eval()`-ed into existence from JSON.
  * Collections must be pleasant to use, even relative to JRE collections. If developers are still tempted to use JRE collections, we could end up with the worst case of duplicating collection code.
  * When none of the other goals are compromised, individual methods on collections should be source-compatible with JRE collections. This simplifies porting existing code to use the new collections and keeps methods as familiar as possible.

## Non-goals
  * We are knowingly sacrificing true compatibility and interop with existing JRE collections. For example, we do not intend to implement the JRE `Map`, `List`, or `Set` interfaces.
  * There is no particular goal of using the same collections implementation for both the client and the server. If we need to use super-source or deferred binding, so be it.
  * Until the GWT compiler can optimize it (which it cannot to date), the new collections will not support the Java enhanced 'for' loop syntax that uses Iterable/Iterator. We do believe such optimizations are possible and will be added, however, at which time, these collections will be retrofitted to implement Iterable.

## Introduction
_Note: The collections described here are similar in several respects to Scala 2.8 collections, and where possible, terminology is borrowed to give consistent names to the similar concepts._

We begin by acknowledging that the design of anything as fundamental as collection types is bound to be contentious, so the plan is to start small and establish some patterns for simple, high-value collections: 'Array' 'Map' and 'Set'.

## Principles
_Assertions, not exceptions._ Assertions, when turned off, can be compiled away. Code written to defensively check for and throw exceptions is very hard to optimize. Development mode (was hosted mode) always has assertions on, so any well-tested code ought to execute assertions during normal testing.

_Types and methods must be used properly to perform properly._ Lightweight collections are in no way defensive. Improper use generates an assertion failure or, if assertions are disabled, undefined behavior. The corollary is that the API can be designed for maximum efficiency.

_Bending the type system is okay if it's sufficiently hard to detect._ If there are cases in which quietly cross-casting JSOs would be the most efficient thing to do, so it shall be done. However, in such cases, it should be a goal to do "the right thing" in hosted mode (e.g. actually have checked casts) but then silently slip in the trickier cross-casting version during a web mode compile.

_Hosted mode collections implementations can differ, if necessary, from web mode._ This follows from the previous point but is also practical — because the development mode browser plug-in (was OOPHM) is based on IPC, it may simply be harmfully slow to implement hosted mode collections using JSNI.

_Publish and guarantee time complexity for various operations._ That is, publish via documentation and enforce empirically by monitoring automated benchmarking tests.

_Keep implementations locked down._ Because it's very tricky to get the implementations for these sorts of collections just right, the plan is to minimize ways in which there could be confusion about the implementation of a given collection type. More concretely, the key types should be classes rather than interfaces so that their implementation can be restricted to the package in which they are defined. If developers cannot trust that a given collection instance is truly optimal and maintains time complexity guarantees -- as might be the case if collections types were interfaces which could have been implemented "anywhere" -- then they may feel compelled to write defensive code. By locking the implementations down, developers can rely on the behavior of a given collection type.

## Collections

```
/**
 * The single starting point for interacting with collections from scratch.
 * All collection instances are originally obtained via a method in this class.
 * Using a factory method rather than a constructor for, say, MutableArray<E> 
 * provides convenient inference of the collection's generic type and also provides
 * a way to satisfy the requirement that JSOs cannot have exposed constructors.
 */
public class CollectionFactory {
  public <E> MutableArray<E> createMutableArray();
  public <E> MutableArray<E> createMutableArray(int size);
  public <E> MutableArray<E> createMutableArray(int size, E fillValue);
  public <K,V> MutableMap<K,V> createMutableMap();
  public <K,V> MutableMap<K,V> createMutableMap(Relation<Object, String> adapter);
  public <E> MutableSet<E> createMutableSet();
  public <E> MutableSet<E> createMutableSet(Relation<Object, String> adapter);
}

/**
 * Used to encapsulate the logic to convert an arbitrary object into a 
 * representative String. MutableMap and MutableSet use these to extend
 * the underlying string-based map and set.
 */
public interface Relation<I,O> {
  O applyTo(I value);
}

/**
 * Used to facilitate iteration over map values and set elements.
 */
public interface Action<T> {
  void applyTo(T element);
}
```

### Array, MutableArray, and ImmutableArray

```
/**
 * A "root collection" that provides a read-only interface to an array 
 * whose contents may or may not be mutable, making it possible to create read-only
 * library algorithms that work with either mutable or immutable arrays without
 * making calling code paranoid about whether the library routine might mutate it,
 * thus preventing the desire to do defensive wrapping or copying.
 */
public class Array<E> {
  @ConstantTime public E get(int index);
  @ConstantTime public int size();
}

/**
 * A "mutable collection" that is very similar to ArrayList 
 * (and C++ vector, and many others). It has the ability to efficiently 
 * "freeze" itself into an ImmutableArray.
 */
public final class MutableArray<E> extends Array<E> {
  // Appends an element to the end of the array
  @ConstantTime public void add(E elem);
  
  // Removes all elements of the array
  @ConstantTime public void clear();
  
  // Creates an immutable array based on the state of this instance.
  // The returned array may or may not have the same identity as this.
  // After this call, this instance may no longer be used as a MutableArray instance
  // (assertions double-check that no subsequent modifications are made).
  @ConstantTime public ImmutableArray<E> freeze();

  // Changes the element at a particular index
  @ConstantTime public void set(int index, E elem);
  
  // Inserts an element into the specified index, lengthening the array by 1
  @LinearTime public void insert(int index, E elem);
  
  // Removes the element at the specified index, shortening the array by 1
  @LinearTime public void remove(int index);

  // Changes the size of the Array filling the new elements with fillValue
  @LinearTime public void setSize(int size, E fillValue);
}

/**
 * An "immutable collection" that makes that iron-clad guarantee that code 
 * holding a reference to an instance can be certain that its 
 * contents will not change over time.
 */
public class ImmutableArray<E> extends Array<E> {
  // Only inherited members
}
```

### Map, MutableMap, and ImmutableMap

```
/**
 * Provides a read-only interface to a map whose contents may or may not be 
 * mutable by other actors.
 */
public class Map<K,V> {
  public boolean containsKey(K key);
  public V get(K key);
  public boolean isEmpty();

  // Provides a way to efficiently iterate over the values in the map
  public forEach(Action<E> function);
}

/**
 * A mutable map. Its underlying support is a map with String keys. An
 * "adapter" (an instance of Relation<Object,String>) object is provided
 * upon its creation to support conversion of keys of type K to Strings.
 */
public class MutableMap<K,V> extends Map<K,V> {
  @LinearTime public void clear();
  @ConstantTime public boolean containsKey(K key);

  // Similar to Array.freeze(), provides a read-only version of the map
  // guaranteed to never change.
  @ConstantTime public ImmutableMap<K,V> freeze();
  @ConstantTime public void put(K key,V value);
  @ConstantTime public void remove(K key);
}

/**
 * A map whose contents cannot change.
 */
public class ImmutableMap<K,V> extends Map<K,V> {
  // Only inherited members
}
```

### Set, MutableSet, and ImmutableSet

```
/**
 * A read-only interface to a set whose contents may or may not be 
 * mutable by other actors.
 */
public class Set<E> {
  @ConstantTime public boolean contains(Object element);
  @LinearTime public boolean containsAll(Set<E> other);
  @LinearTime public boolean containsSome(Set<E> other);
  @ConstantTime public boolean isEmpty();
  @LinearTime public boolean isEqual(Set<E> set);

  // Provides a way to efficiently iterate over the values in the set
  public forEach(Action<E> function);
}

public class MutableSet<E> extends Set<E> {
  @ConstantTime public void add(E elem);

  // Union
  @LinearTime public void addAll(Set<E> other);

  @LinearTime public void clear();
  
  // Similar to Array.freeze(), provides a read-only version of the set
  // guaranteed to never change.
  @ConstantTime public ImmutableSet<E> freeze();

  // Intersection
  @LinearTime public void keepAll(Set<E> other);

  // Substraction
  @LinearTime public void removeAll(Set<E> other);
}

/**
 * A set whose contents cannot change.
 */
public class ImmutableSet<E> extends Set<E> {
  // Only inherited members
}
```

## Examples of Intended Usage

### Arrays
```
public <E> void printEach(Array<E> arrToPrint) {
  for (int i = 0, n = arrToPrint.size(); i < n; ++i) {
    Window.alert(arrToPrint.get(i).toString());
  }
}

public MutableArray<String> makeSnackArray() {
  MutableArray<String> a = createMutableArray();
  a.add("apple");  
  a.add("banana");
  a.add("coconut");
  a.add("donut");
  return a;
}

public void exampleOne() {
  MutableArray<String> a = makeSnackArray();
  printEach(a);
}

public void exampleTwo() {
  MutableArray<String> ma = makeSnackArray();
  ImmutableArray<String> ia = ma.freeze();

  // We can't touch 'ma' any more, but we can
  // confidently hand out aliases all over the place.
  new MyView1(ia).show();
  new MyView2(ia).show();
  new MyView3(ia).show();
}
```

### Maps

```
public <K,V> void printOne(Map<K,V> mapToPrint, K key) {
  Window.alert(mapToPrint.get(key).toString());
}

Relation<Object,String> integerAdapter = new Relation<Object, String>() {
  public String applyTo(Object element) {
    if (element == null || !(element instanceof Integer)) {
      return null;
    }
    return "__" + element.toString();
  }
};

public MutableMap<Integer,String> makeSparseSnackMap() {
  MutableMap<Integer,String> map = CollectionFactory.createMap(integerAdapter);
  map.put(123456, "apple");
  map.put(4321, "orange");
  map.put(0, "lemon");
  return map;
}

public void exampleThree() {
  MutableMap<Integer,String> m = makeSparseSnakMap();

  printOne(m, 123456);
  if (m.containsKey(0)) {
    printOne(m, 0);
  }
}

public void exampleFour() {
  MutableMap<Integer,String> mm = makeSparseSnakMap();

  ImmutableMap<Integer,String> im = mm.freeze();

  // We can't touch 'mm' any more, but we can
  // confidently hand out aliases all over the place.
  new MySnackTableView1(im).show();
  new MySnackTableView2(im).show();
  new MySnackTableView3(im).show();
}
```

### Sets

```
Action<String> printSnack = new Action<String>() {
  public void applyTo(String value) {
    Window.alert(value.toString());
  }
}

public <T> void printAll(Set<T> setToPrint) {
  setToPrint.forEach(printSnack);
}

Relation<Object,String> caseInsensitiveAdapter = new Relation<Object, String>() {
  public String applyTo(Object element) {
    return (element == null)? null : ("__" + element.toString().toUpperCase());
  }
};

public MutableSet<String> makeSnackSet() {
  MutableSet<String> set = CollectionFactory.createSet(caseInsensitiveAdapter);
  set.add("apple");
  set.add("Orange");
  set.add("LEMON");
  return set;
}

public MutableSet<String> makePreferredSnackSet() {
  MutableSet<String> set = CollectionFactory.createSet(caseInsensitiveAdapter);
  set.add("Apple");
  set.add("Pear");
  return set;
}

public MutableSet<String> makeYuckySnackSet() {
  MutableSet<String> set = CollectionFactory.createSet(caseInsensitiveAdapter);
  set.add("lemon");
  set.add("grapefruit");
  return set;
}

public void exampleFive() {
  MutableSet<String> ms = makeSnackSet();

  printAll(ms);
}

public void exampleSix() {
  MutableSet<String> someSnacks = makeSnackSet();
  MutableSet<String> great = makePreferredSnackSet();
  MutableSet<String> bad = makeYuckySnackSet();

  ImmutableSet<String> frozenSnacks = someSnacks.freeze();

  // We can't touch frozen sets any more, but we can
  // confidently hand out aliases all over the place.
  great.keepAll(frozenSnacks);
  bad.keepAll(frozenSnacks);

  new AllView(frozenSnacks).show();
  new PreferredView(great.freeze()).show();
  new DislikesView(bad.freeze()).show();
}
```