# Background

The GWT compiler does an extremely good job today of optimizing Java in compiling to Javascript,
which is all the more surprising and amazing, since it achieves most of this using non-traditional
algorithmic techniques, and without even doing many of the classic optimizations that many people
learn in academic compiler courses.

The purpose of this document is to explore some of these classical techniques and see which if any
might be candidates for making improvements in the compiled output, both in size and speed, as well
as explore new non-traditional techniques.

# Classic Techniques

The list classical techniques we'd like to explore, prerequisites that are needed,
and how they might be implemented in the current compiler infrastructure:

  * Intra-procedural/intra-block optimization
    * Dead Code Elimination
    * Copy Propagation
    * Constant Folding
    * Common Subexpression Elimination / Partial Redundancy Elimination
    * Local variable allocation (as if they were registers)
    * Loop hoisting / Loop Induction
  * Prerequisites
    * Linearized AST
    * Control Flow Graph
    * Data Flow Analysis / Liveness Analysis
      * SSA form conceptually easier to deal with
  * Others Techniques
    * Object Inlining
    * Type-Cloning
    * Escape Analysis

# Intra-procedural optimization

GWT already does [dead code elimination](http://en.wikipedia.org/wiki/Dead_code_elimination), [constant folding](http://en.wikipedia.org/wiki/Constant_folding), and [copy propagation](http://en.wikipedia.org/wiki/Copy_propagation), but it does so without
regard to [control flow](http://en.wikipedia.org/wiki/Control_flow_graph) and [liveness information](http://en.wikipedia.org/wiki/Liveness_analysis), so for example, the following fragment:

```
int x;
x = 2;
x = 4;
Window.alert(""+x);
```

will not detect that the assignment `x = 2` is dead code. A block level optimizer with [dataflow information](http://en.wikipedia.org/wiki/Data_flow_analysis)
at hand will be able to spot more optimization opportunities. A classic example is heavy use of the [http://en.wikipedia.org/wiki/Builder_pattern builder
pattern] with method chaining:

```
Builder b = new Builder();
b.setA(10).setB(20).setC(30);
```

The GWT compiler will treat the builder methods as static functions and rewrite this as:

```
Builder b = new Builder();
setC(setB(setA(b, 10), 20), 30);
```

the compiler is currently unable to inline these functions. If they were rewritten as:

```
Builder b = new Builder();
t1 = setA(b, 10);
t2 = setB(t1, 20);
t3 = setC(t2, 30);
```

then the compiler could inline them, however it leaves many superfluous assignments, e.g.

```
b.A = 10;
t1 = b;
t1.B = 20;
t2 = t1;
t2.C = 30;
t3 = t2;
```

when the ideal code is:

```
b.A = 10;
b.B = 20;
b.C = 30;
```

Performing these optimizations may be easier to do if statement blocks are flattened into a form of [three-address code](http://en.wikipedia.org/wiki/Three_address_code), so that liveness ranges are easier to compute.

## Common Subexpression Elimination

GWT currently does not perform [CSE](http://en.wikipedia.org/wiki/Common_subexpression_elimination). It is unclear how much of a difference it will make for well written programs, but some papers make claims for non-trivial benefits in scientific computing. However, in large code bases, duplications probably do arise, and there might be cases where repeating an expression may lead to better readability. There may also be unavoidable common subexpressions that occur as a by-product of other optimizations (like loop induction).

Probably the most well known way of doing this is with the [Value Numbering technique](http://en.wikipedia.org/wiki/Global_value_numbering).

## Local Variable Allocation

While Javascript is not machine code and doesn't have registers, the number of live temporaries have an effect on code size, on the gzip/deflate compression dictionary, on the obfuscator, and perhaps even the Javascript VM. The intention here is to treat Javascript locals as if they were machine registers (on an infinite register machine) and use traditional [register allocation](http://en.wikipedia.org/wiki/Register_allocation) techniques to minimize the number of locals used in each scope. As an example:

```
String s = "Hello";
Window.alert(s);
String s2 = "World";
Window.alert(s2);
```

could be transformed into:

```
String s = "Hello";
Window.alert(s);
s = "World";
Window.alert(s2);
```

Moreover, the need to introduce temporaries to linearize the AST blocks and optimize examples like the Builder pattern, will increase code size unless there is an extra pass to remove unneeded temporaries.

## Loop Hoisting / Loop Induction

[Loop Hoisting](http://en.wikipedia.org/wiki/Loop-invariant_code_motion) would find loop-invariant expressions and move them out of the body of the loop. Loop Induction finds variables which increase according to a linear function of the loop index, and remove the loop index from the equation.

Hoisting example:
```
for(int i=0; i<foo.size(); ++i) {
  String msg = locale.getMessage();
  Window.alert(msg + foo.get(i));
}
```

after hoisting:
```
int n = foo.size();
Sring msg = locale.getMessage();
for(int i=0; i<n; i++) {
  Window.alert(msg + foo.get(i));
}
```

Induction example:
```
double matrix[] = new double[ROWS*COLS];
for(int i=0; i<ROWS; i++) {
  int j = i*COLS + 2;
  Window.alert("Column 2 of Row "+i+" is: "+matrix[j]);
}
```

after induction:
```
double matrix[] = new double[ROWS*COLS];
int j = - COLS + 2;
int inc = COLS; 
for(int i=0; i<ROWS; i++) {
  j = j + inc;
  Window.alert("Column 2 of Row "+i+" is: "+matrix[j]);
}
```

This may or may not be faster on JS VMs. It does reduce the number of referenced variables in the
expression for j and changes a multiplication into an addition. There are a number of other [loop optimizations](http://en.wikipedia.org/wiki/Loop_optimization) that could be candidates, such as loop fusion.

# Prerequisites

Many compilers have been traditionally written to turn an AST into a flat intermediate representation like three address code, for example the GNU C compiler. Recently however, GCC introduced a new intermediate representation based on preserving high level AST structure, called GENERIC/GIMPLE. [http://gcc.fyxm.net/summit/2003/GENERIC%20and%20GIMPLE.pdf GENERIC and GIMPLE A new tree
representation for entire functions] Since GWT's current optimizations are based strictly on ASTs and not a low-level IR, it makes sense to follow in GCC's footsteps at this point and avoid a switch to a low level IR which would require a large re-engineering effort in the compiler.

## Linearize AST
In order to perform many of these optimizations, a control flow graph is usually needed, that is, you need
to group a sequence of statements into [basic blocks](http://en.wikipedia.org/wiki/Basic_block), along with connectivity information, but this is difficult to do with the current AST structure, where a node within a given block, may in fact, represent several basic blocks, with no easy way to overlay a control flow structure.

Consider:
```
while(a + b + c < MAX && j < retries) {
  j++;
  a = b >  c ? c : b;
  b = (b * 3 + 1) % 2;
  c = c/2;
}
```

The JWhileStatement will be the root node, but in fact, the first thing to be executed is 'a+b', and due to short-circuiting, control flow within the conditional is actually two blocks. This loop can be visualized as:

```
// BLOCK 1
t1 = a + b;
t2 = t1 + c;
LOOP:;
tcond = t2 < MAX; // BLOCK2
if(tcond) {
  tcond2 = j < retries; // BLOCK3
  if(tcond2) {
      j++;  // BLOCK4
      tcond3 = b > c;
 if(tcond3) 
        a = c; // BLOCK5
      else
        a = b; // BLOCK6
      b = (b * 3; // BLOCK7
      b = b + 1;
      b = b % 2;
      c = c/2;
      goto LOOP;
  }
}
```

GCC calls this _GIMPLE_ and refers to the process as _gimplifying_.

Linearizing the AST could have detrimental effects on output code as well as other optimizations, since some of the original source structure will be obscured. To guard against a potentially bad payoff, the linearizing can be done in two phases.

### Phase 1

No control flow graph. Only flatten JExpressions by introducing temporaries. Do not flatten loops, conditionals, or other flow control structures. This will permit many of the intra-block optimizations to be made, but won't enable useful cross-block dataflow. It chould have some positive benefit for Builder patterns which usually consist of chained methods with no conditionals, which if inlined fully, would already be a basic block.

### Phase 2

Flatten for, while, do/while, if/else, switch/case, and other high level conditional statements. This may entail rewriting a flow control construct as a different high level construct. If possible, provide an unflatten routine which can recover the original high level construct (i.e. for vs while)

## Control Flow Graph

After linearizing the AST, it should be straight forward to build traditional flow control graph of basic blocks. The representation is another matter. One way of doing it is to build separate CFGNodes, composed of CFGBlocks which contain JNodes. This is yet another, parallel datastructure, and may vastly increase memory usage, although the CFGs are only needed per-procedure and can be thrown away.

Another possibility is to introduce a variant of JBlock, a JCFGBlock which exists in the Java AST, and can be used to group existing AST nodes, so that JBlocks are split up and rewritten as JCFGBlocks.

JCFGBlocks would contain additional methods, fields, and interfaces to track in/out connections, dataflow information, as well as providing visitor methods for walking over a CFG.

## Data Flow Analysis

Once the tree is linearized into basic blocks of roughly three-address-code comparable AST nodes, one of the first pieces of information that can be gathered is the lifetime of local variables. That is, the range of statements within a block in which the variable has it's value defined, up to the last time it is referenced. This analysis is made more difficult if variables are not immutable, for example:

```
int x = 20
int y = x * 3 + 10;
Window.alert(y + x);
x = 30;
Window.alert(y * 2 + x);
```

In this example, y's live range is between line 2 and line 5. However, x has two live ranges. The first occurs between line 1 and line 3. The second between line 4 and line 5, because x is redefined on line 4.

### Static Single Assigment (SSA) Form

To combat and make liveness analysis easier, we treat subsequent definitions of a variable as a new variable. Thus, we can rewrite the above example:
```
int x = 20
int y = x * 3 + 10;
Window.alert(y + x);
x2 = 30;
Window.alert(y * 2 + x2);
```

Now it is clear that a variable's live range begins at its definition, and ends on the last line that references it. A wrinkle occurs if there is a conditional:

```
x = 5;
if(expr) {
 x = 10;
}
else {
 x = 20; 
}
Window.alert(x);
```

if we apply our rule that every definition creates a new name:
```
x = 5;
if(expr) {
  x2 = 10;
}
else {
  x3 = 20;
}
Window.alert(x2 if the then clause, x3 if the else clause);
```

We have no way of representing the choice of one variable or the other in the AST node depending on which path was taken. We can invent one, let's call it JPhiNode and put it into the AST, which contains a list of all versions of x reaching a certain point;
```
x = 5;
if(expr) {
  x2 = 10;
}
else {
  x3 = 20;
}
Window.alert(phi(x2, x3));
```

This is a place holder for the compiler. The compiler can't know which path was taken until runtime, and phi is not an executable statement, but later on when optimizations are over, we can convert out of SSA form, which means translating those phi functions back to real code. For more on SSA and phi-functions see, [Converting to SSA](http://en.wikipedia.org/wiki/Static_single_assignment_form#Converting_to_SSA)

## More Data Flow Analysis

### Register Allocation / Interference Graph

With live-range information, we can prune unused variables better, we can also perform 'register allocation' by figuring out which live-ranges overlap, and which do not, and using that information to minimize the number of local variables used by a function. As mentioned earlier, this should help out somewhat with the obfuscator and gzip. See [recent developments in register allocation](http://en.wikipedia.org/wiki/Register_allocation#Recent_developments) for some other potential techniques.

### Dominators

Given a block B\_j, a block B\_i [\*dominates\*](http://en.wikipedia.org/wiki/Dominator_(node)) B\_j if all paths from the start of the program reaching B\_j have to pass through B\_i. If B\_i dominates B\_j, and no block B\_x exists which dominates B\_j, but is dominated B\_i, we call B\_i an **immediate dominator**.

Dominance information enables a number of optimizations. For GWT in particular, dominance information can provide better pruning for clinits, as well as type-tightening. In particular, if clinitA occurs in a dominator of B\_j, it may be pruned from B\_j since it is guaranteed that to reach B\_j, clintA must have already be executed earlier.

In the case of type-tightening, types may be tracked between blocks. If a variable in block B\_j is live-in from some block B\_i which dominates B\_j (that is, it was defined/assigned in block B\_i), and the type of the variable is known to be type T, then it may be tightened to T.  Currently, GWT only does this analysis at the global level and does not track type-flow through multiple stack frames.

That said, these analyses require a whole-program CFG, rather than just a procedural one.

# Other Techniques

## Object Inlining

Object Inlining is a technique for eliminating the allocation of an object, by inlining all of its fields into another object, or as local scalar variables on the stack. Consider the following code:

```
public class Point {
 private int x, y;
}

public class Rect {
  Point topLeft;
  Point bottomRight;

  public int area() {
     return (bottomRight.x - topLeft.x) * (bottomRight.y - topLeft.y);
  }
}
```

For demonstration purposes, inline the fields of the `Point` class by prefixing them with the field name:
```
public class Rect {
  int topLeft_x, topLeft_y;
  int bottomRight_x, bottomRight_y;

  public int area() {
     return (bottomRight_x - topLeft_x) * (bottomRight_y - topLeft_y);
  }
}
```

This eliminates the existence of the Point class. If Point had any methods, they could be inlined as specializations (cloned for each instance), and hopefully pruned after further inlining in expressions, e.g.


```
public class Point {
 private int x, y;
 public int getX() { return x; }
 public int getY() { return y; }
}

public class Rect {
  Point topLeft;
  Point bottomRight;

  public int area() {
     return (bottomRight.getX() - topLeft.getX()) * (bottomRight.getY() - topLeft.getY());
  }
}
```

Consider inlining those methods the same as the fields, by prefixing them. Polymorphism would be problematic to handle, requiring inlining the union of potential type's fields as a sort of discriminated union, and probably not worth the effort.

In general, it's probably best to avoid object inlining except where the concrete type is known and all methods are effectively static. This roughly isomorphic to Overlay Types.

A complication arises if `Point` escapes scope, either leaks out of the class through a method, or leaks from local stack scope.

```
public class Rect {
  public void draw(Canvas c) {
    c.plot(topLeft); // BAD!
  }

  public Point getTopLeft() {
    return topLeft; // UH OH!
  }
}
```

We can handle this in one of two ways: construct a `new Point(topLeft_x, topLeft_y)` and pass that value. Or, if this situation is detected, back out of the inlining optimization. The latter is probably safer and would avoid pathological performance problems (such as someone calling getTopLeft() in a loop) This leads to the following rule:

_An object A may not be inlined into an Object B if any method exists on B which passes a reference to A or returns a reference to A_

Additionally, objects may be inlined into local scope, consider:

```
public void someMethod(Rect r) {
  Point p = r.getTopLeft();
  Window.alert("Coordinates are "+p.x+", "+p.y);
}
```

This could be written as
```
  var p_x = r.topLeft_x;
  var p_y = r.topLeft_y;
  Window.alert("Coordinates are "+p_x+", "+p_y);
```

Which provides an exception to the above rule:
_An object A may not be inlined into an Object B if any method exists on B which passes a reference to A or returns a reference to A_, except in cases where the reference may be inlined into the caller/callee.

A reference could be inlined into the callee in cases where the object is inlined, and no other callers invoke the method with non-inlined versions. Consider:

```
public class PointUtil {
  public static boolean equals(Point p1, Point p2) {
     return p1.x == p2.x && p1.y == p2.y;
  }
}
```

If `equals` is never invoked from any callsite with a non-inlined version, then the method may be rewritten as:
```
public class PointUtil {
  public static boolean equals(int p1x, int p1y, int p2x, int p2y) {
     return p1x == p2x && p1y == p2y;
  }
}
```

## Type Cloning
(via Bruce)
For every unique (lexical) occurrence of an allocation of type T, clone T into T' as if it were an independent type, but arrange that T' is indistinguishable from T w.r.t. casts, instanceof, and getClass(). In other words, you could never determine at runtime that the object wasn't actually a genuine T. Now, optimize as normal. To see how this could play out:

```
class BasicList<E> {
  private E[] elems;

  @SuppressWarnings("unchecked")
  public void add(E elem) {
    if (elems == null) {
      elems = (E[]) new Object[] {elem};
    } else {
      E[] newElems = (E[]) new Object[elems.length + 1];
      System.arraycopy(elems, 0, newElems, 0, elems.length);
      newElems[newElems.length - 1] = elem;
      elems = newElems;
    }
  }

  public int size() {
    return elems == null ? 0 : elems.length;
  }
  
  public Iterator<E> iterator() {
    final int n = size();

    return new Iterator<E>() {
      private final int i = 0;

      public boolean hasNext() {
        return i < n;
      }

      public E next() {
        return elems[i];
      }

      public void remove() {
        // nothing
      }
    };
  }
}
```

```
// Then in some other code, you could write...
void foo() {
  BasicList<String> b = new BasicList<String>();
  for (int i = 0, n = b.size(); i < n; ++i) {
    // Removed by dead code elim.
  }

  for (String s: b) {
    // Also removed by dead code elim.
  }
}
```
In the above, `b` is of type `BasicList'` (a clone of `BasicList` that can be independently optimized). Thus, you can see that `BasicList'` does not have the `add()` method called, which means that, for this particular use of the type, the `elems` field will always be `null`. Consequently, `size()` can be statically known to return `0`. Similarly, the iterator's `hasNext()` method can be statically shown to always return false in the for/each loop.

The key point is that the usage of a type in one context would not affect how the type itself could be optimized in other contexts. As it stands today, we can't do the above optimization if anywhere in the program someone calls `BasicList#add()`. A great practical example is Widgets firing events. You want widgets to be **able** to fire events **if** someone actually wants to listen to them. In the cases where you can see that there are no listeners to a widget, you'd really like the entirely of the even-firing infrastructur to completely disappear.

### Global Object Interning/Hoisted Inlining

Inspired by Joel's attempts to optimize Java Enums, consider that we have a class `Foo`

```
public class Foo {
  final int field1;
  final String field2;

  public Foo(int x, String y) { this.field1=x; this.field2=y; }
  public int getField1() { return field1; }
  public String getField2() { return field2; }
}
```

and this class is provably only ever instantiated with compile time literal constants and contains no ability to mutate these fields.

```
new Foo(42, "Adams");
new Foo(99, "Agent");
new Foo(5, "Johnny");
new Foo(10100, "Google");
```

Then the compiler may globally intern/hoist these field constants:
```
public class FooHoist {
  public static int[] field1= {42, 99, 5, 10100};
  public static String[] field2 = {"Adams", "Agent", "Johnny", "Google"};
}
```

And any reference to `Foo` may now be replaced with an ordinal index into `FooHoist`. e.g. `new Foo(5, "Johnny");` becomes  `2`.

This would allow the entire classtype for `Foo` to be pruned in many circumstances, the chief example being Enum classes. In the case where `Foo` is polymorphic, you need a separate dispatch table, but you can still eliminate the type and it's subtypes. In cases where Foo is passed and upcast to Object or some supertype (say for collection usage), it may need a shim to wrap the ordinal, but this shim might be reusable across all such interned objects.

This optimization is mostly useful for Enums and Type-Safe enums.

## Semantic Techniques

Let us call a method semantic, if it is recognized by the compiler by name, and the compiler has a heuristic for evaluating the same function, discarding the original implementation.  Semantic techniques are used by a number of compilers to replace operations with intrinsically faster ones. Consider the Java library function `Math.sin()`. While the library could theoretically have an implementation of sin() using table lookup or power series expansion, in some architectures, there is a native CPU instruction for computing `sin`, and therefore, the JIT compiler can in many cases, detect the call to Math.sin(), and replace it with an inlined instruction. Let us call this technique _semantic inlining_.

Given the prior  technique of _Type Cloning_, it may be possible to speed up JRE collections significantly. Let's see how. Consider the following code:

```
HashMap<String, String> map = new HashMap<String, String>();
map.put("apples", "Ray");
map.put("grapes", "Bruce");
map.put("oranges", "Scott");
for(String key : map.keySet()) {
   Window.alert(map.get(key) + " likes " + key);
}
```

What we would like to see as output is:

```
map = {};
map['apples']="Ray";
map['grapes']="Bruce";
map['oranges']="Scott";
for(var key in map) {
  $wnd.alert(map[key] + ' likes ' + key);
}
```

If we recognize that the cloned type T is HashMap<String, String>, and that the method entrySet(), values(), remove(), and others are never invoked, might it be possible to replace the implementation of HashMap<String,String> for T with a simpler, pre=optimized one based on the needs of a simple string dictionary lookup?

Furthermore, the pattern `for(key : map.keySet())` is extremely common and has a corresponding efficient Javascript representation. Perhaps this can be recognized too and replaced with a JSNI method call/JSNI fragment.

This applies equally well to `ArrayList`:
```
ArrayList list = new ArrayList();
list.add("foo");
list.add("bar");
list.add("baz");
for(String word : list) {
 Window.alert(word);
}
```

which could be written in Javascript as:

```
list = [];
list[list.length] = "foo";
list[list.length] = "bar";
list[list.length] = "baz";
for(var i=0; i<list.length; i++) {
   $wnd.alert(list[i]);
}
```

Since many of the methods of ArrayList are unused in this circumstance, and java.util.Iterator is never used directly, a semantically replaced/inline version could avoid JRE collections altogether.