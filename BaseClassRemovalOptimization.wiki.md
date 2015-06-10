# Introduction

Seems like there are some opportunities to more aggressively remove base classes when they really aren't needed at runtime.

# Details

Here's a trivial example:
```
class A {
  String aState;
  A(String aState) {
    this.aState = aState;
  }
}

class B extends A {
  String bState;
  B(String aState, String bState) {
    super(aState);
    this.bState = bState;
  }
}

B b = new B("a", "b");
Window.alert(b.aState + b.bState);
```

This currently gets compiled to something like this:

```
var _;
function Object_0(){
}

_ = Object_0.prototype = {};
_ = String.prototype; // Why is this here!!??

function A(){
}

_ = A.prototype = new Object_0();
_.aState = null;
function $B(this$static, aState, bState){
  this$static.aState = aState;
  this$static.bState = bState;
  return this$static;
}

function B(){
}

_ = B.prototype = new A();
_.bState = null;

function init(){
  var b;
  b = $B(new B(), 'a', 'b');
  $wnd.alert(b.aState + b.bState);
}

```

You will notice there are two completely empty base classes in the compiled output that serve no purpose. It seems like those could be easily eliminated. Also, the class representation of B doesn't really seem necessary either. I'm wondering why this couldn't be transformed directly to:

```
function init() {
  var b;
  b = {aState : 'a', bState : 'b' };
  $wnd.alert(b.aState + b.bState);
}
```

# Slightly Modified Proposal

With the optimizations described in ClassSetupAndInstantiationOptimization, the output code would look more like this (I've pulled out the reference to String.prototype, which is just a small leftover from having such a small app, with no String methods called):

```
var _;

function Object_0() {}
_ = Object_0.prototype = {};

function Hello$A() {}
_ = Hello$A.prototype = new Object_0;

function Hello$B(aState, bState){
  this.aState = aState;
  this.bState = bState;
}
_ = Hello$B.prototype = new Hello$A;

function init(){
  var b;
  b = new Hello$B('a', 'b');
  $wnd.alert(b.aState + b.bState);
}
```

This case still leaves useless `Hello$A` and `Object_0` classes (to be fair, Object\_0 will almost always be necessary in "real" apps, because you invariably end up with virtual hashCode(), toString(), and equals() methods that can't be stripped).

We should be able to entirely remove any class that is never instantiated, and has no fields or methods (either because the class never **had** fields or methods, or because they were stripped by the compiler). That would leave us with this:

```
var _;

function Hello$B(aState, bState){
  this.aState = aState;
  this.bState = bState;
}
_ = Hello$B.prototype = {};

function init(){
  var b;
  b = new Hello$B('a', 'b');
  $wnd.alert(b.aState + b.bState);
}
```

As for whether we could take this a step further and eliminate the constructor altogether, I think this is unlikely in practice, because (as I mention above) all real apps will have virtual methods on Object that can't be eliminated.

It's also worth noting that, though we could technically get rid of `Hello$B`'s prototype assignment, there's no point in practice, because it will normally be set to `new Object`. We can and should, however, get rid of the unnecessary `_ = ` part when there are no subsequent prototype assignments.

# Performance Impact

It can only be faster to **not** setup extra classes :)