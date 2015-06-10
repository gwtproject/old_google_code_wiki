# Introduction

The GWT compiler generates standard Javascript-style class hierarchies with prototype chains used to represent parent-child relationships. This leads to a lot of generated code like this:

```
function Sub() {}
_ = Sub.prototype = new Super();
_.member1 = value1;
...
```

This could be made more terse with a structure like the following:

```
function $(sub, super) {
  return sub.prototype = new super();
}

function Sub() {}
_ = $(Sub, Super);
_.member1 = value1;
...
```

# Performance Impact

Unfortunately, moving function invocations into the prototype chaining process is slower on every browser we've tested. The speed difference ranges from 4% on Chrome to 30% on Firefox 3. On the Android browser, it's slower by a whopping 45%. We need a more detailed investigation to determine how much of a practical difference it makes on real code, but because the speed difference affects application startup, we must tread carefully.

# Alternative Formulation

An alternative formulation proposed recently would take the following form:

```
function $(superClass, members){
  var result = function() {};
  result.prototype = members;
  members.__proto__ = superClass.prototype;
  return result;
}

var myClass=$(superClass, {
  member1: value1,
  ...
});
```

This effectively avoids the `new` expression as well as the series of assignment statements used to initialize a class' members. It reportedly performs just as well as the original formulation. Unfortunately, the proto field doesn't exist on Internet Explorer (at least not as of IE7), so in order to use it we would have to move knowledge of User-Agent into the compiler.