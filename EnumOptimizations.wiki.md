# Introduction

Java's enum semantics require each value to essentially be an anonymous subclass of its enum class. The following example:

```
  enum Foo {
    FOO_0("0"), FOO_1("1") {
      public String getMessage() {
        return "something different";
      }
    }, FOO_2("2");

    private final String msg;

    Foo(String msg) {
      this.msg = msg;
    }

    public String getMessage() {
      return msg;
    }
  }
```

gives some idea of why it's done this way. The GWT compiler currently generates the code for enums following these semantics quite literally, which can lead to some fairly significant bloat. In this case, the initialization for the Foo enum values alone looks like this:

```
  FOO_0 = $Hello$Foo(new Hello$Foo(), 'FOO_0', 0, '0');
  FOO_1 = $Hello$Foo$1(new Hello$Foo$1(), 'FOO_1', 1, '1');
  FOO_2 = $Hello$Foo(new Hello$Foo(), 'FOO_2', 2, '2');
```

along with a class definition for Foo like this:

```
function $Hello$Foo(this$static, enum$name, enum$ordinal, msg){
  $clinit_6();
  this$static.ordinal = enum$ordinal;
  this$static.msg = msg;
  return this$static;
}

function getMessage_0(){
  return this.msg;
}

function Hello$Foo(){
}

_ = Hello$Foo.prototype = new Enum();
_.getMessage = getMessage_0;
_.typeId$ = 0;
_.msg = null;
```

and a class definition for Foo$1 like this:

```
function $Hello$Foo$1(this$static, enum$name, enum$ordinal, $anonymous0){
  $clinit_5();
  this$static.ordinal = enum$ordinal;
  this$static.msg = $anonymous0;
  return this$static;
}

function getMessage(){
  return 'something different';
}

function Hello$Foo$1(){
}

_ = Hello$Foo$1.prototype = new Hello$Foo();
_.getMessage = getMessage;
_.typeId$ = 0;
```

# Improvements

Clearly it's a lot smaller when obfuscated, but there's plenty of room for improvement here. It's probably true that most enums, most of the time, are semantically equivalent to their ordinals (i.e., they just add a bit of type safety on top of using integers). Also, it's not possible to extend enums, so the number of cases where the generated code has to deal with virtual dispatch is somewhat limited.

Giant TODO: How can we make all this stuff simpler and not break the general case? Ideally, we would compile all enums down to just integers, but virtual dispatch would seem to make this somewhat complicated.

# Open Questions and Ideas

  * Does the Java spec say anything about how enums' default toString() has to behave?
    * If not, we could probably drop all the strings in enum initializers.
    * If so, we could probably still drop them, using something like -XdisableClassMetadata.
  * Could we deal with virtual dispatch on enum methods by simply implementing an ordinal-indexed array of function pointers?


# Performance Impact

Any optimizations that make generated enum code smaller seem likely to be a net performance win, but we'll have to keep a close eye on them to be sure.