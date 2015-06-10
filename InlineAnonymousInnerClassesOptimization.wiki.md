# Introduction

The Java compiler implements anonymous inner classes as specially-named classes that contain copies of final locals and a reference to the outer 'this' context, but are otherwise identical to explicitly-named classes. The GWT compiler currently preserves this behavior, which tends to generate a lot of class definitions in compiled output.

Anonymous inner classes, however, have more limited semantics than normal classes. Most importantly, they cannot be explicitly named in Java code and thus can be instantiated only where they are declared. This bears a striking resemblance to the Javascript practice of creating a function in the context of another function which references the outer function's variables via closure. It is also worth noting that both structures are commonly used in attaching event handlers.

# Details

## A Simple Example

Take the following example of a simple event handler in Java:
```
    btn.addClickHandler(new ClickHandler() {
      public void onClick(ClickEvent event) {
        Window.alert("foo");
      }
    });
```

The compiled output looks like this (in pretty mode, of course; and you're just seeing $addDomHandler below because btn.addClickHandler() has been inlined away):

```
function onClick_0(event_0){
  $wnd.alert('bar');
}

function Hello$1(){
}

_ = Hello$1.prototype = new Object_0();
_.onClick = onClick_0;
_.typeId$ = 10;

function init() {
  $addDomHandler(btn, new Hello$1(), TYPE);
}
```

The synthetic class `Hello$1` cannot be instantiated anywhere else, so giving it a name (it's anonymous after all) serves no purpose, and attaching its properties to its prototype is also unnecessary, as it can't be extended either. So what if we simply produced the following:

```
// Copies p's properties to o. This is defined once for use by all anonymous classes.
function ext(o, p) {
  for (k in p) {
    o[k] = p[k];
  }
}

function onClick_0(event_0){
  $wnd.alert('foo');
}

function init() {
  $addDomHandler(
    b1,
    ext(new Object_0, {
      typeId$: 10,
      onClick: onClick_0
    }),
    TYPE
  );
}
```

Now we simply define and instantiate the anonymous class in one step. But we can take it one step further, because there's no reason to give onClick\_0 a name:

```
function init() {
  $addDomHandler(
    b1,
    ext(new Object_0, {
      typeId$: 10,
      onClick: function(event_0) {
        $wnd.alert('foo');
      }
    }),
    TYPE
  );
}
```

This is getting fairly close to being simply a Javascript function. The main differences are that it needs an Object identity (e.g., if it were placed in a collection), and its onClick method is explicitly named rather than simply being a function object (because other classes that implement ClickHandler might have other methods as well, and the caller can't tell the difference).

## A More Complex Example

So what about anonymous classes that reference the outer 'this', final locals, and have their own fields? Let's try this example from HandlerManager (I've taken some liberties with method names to avoid confusion stemming from inlining):

```
void enqueueAdd(final GwtEvent.Type<H> type, final H handler) {
  defer(new AddOrRemoveCommand() {
    public void execute() {
      registry.addHandler(type, handler);
    }
  });
}
```

This gets compiled into the following Javascript:

```
function $HandlerManager$1(this$static, this$0, val$type, val$handler){
  this$static.this$0 = this$0;
  this$static.val$type = val$type;
  this$static.val$handler = val$handler;
  return this$static;
}

function HandlerManager$1(){
}

function $execute(this$static) {
  $addHandler(this$static.this$0.registry, this$static.type, this$static.handler);
}

_ = HandlerManager$1.prototype = new Object_0();
_.typeId$ = 8;
_.this$0 = null;
_.val$handler = null;
_.val$type = null;
_.execute = $execute;

function $enqueueAdd_0(this$static, type, handler){
  $defer(
    this$static,
    $HandlerManager$1(new HandlerManager$1(), this$static, type, handler)
  );
}
```

Yuck. Let's see what happens when we try to inline `HandlerManager$1`:

```
function $execute() {
  $addHandler(this.this$0.registry, this.type, this.handler);
}

function $addHandler_0(this$static, type, handler){
  $defer(
    this$static,
    ext(new Object_0, {
      typeId$: 8,
      this$0: this$static,
      val$handler: type,
      val$type: handler,
      execute: $execute
    })
  );
}
```

That sucks a bit less. Let's inline that `execute` definition:

```
function $addHandler_0(this$static, type, handler){
  $defer(
    this$static,
    ext(new Object_0, {
      typeId$: 8,
      this$0: this$static,
      val$handler: type,
      val$type: handler,
      execute: function() {
        $addHandler(this.this$0.registry, this.val$type, this.val$handler);
      }
    })
  );
}
```

There are still a couple of fields that are just copies of variables we can reference via closure. Let's ditch those too:

```
function $addHandler_0(this$static, type, handler){
  $defer(
    this$static,
    ext(new Object_0, {
      typeId$: 8,
      this$0: this$static,
      execute: function() {
        $addHandler(this.this$0.registry, type, handler);
      }
    })
  );
}
```

Now we're talking. We still need the this$0 field to get to the outer class' registry field, but this is a lot more terse than what we started with.

# Caveats

I haven't called it out explicitly in these examples, but if the anonymous class had an initializer, it would need to either be inlined into the instantiation site, or called as a separate function after the call to ext(). I suspect this will happen fairly rarely in practice, as not many anonymous classes have explicit initializers -- most of them are just field initializers, which can be inlined easily.

# Performance Impact

I haven't measured this yet, but I'm sure that having to create an extra object and copy its fields over to the anonymous class is slower than just instantiating a normal object. However, there's an upside in that none of these classes have to be setup during application startup -- hopefully the performance effect will simply be to amortize anonymous class setup across the application's runtime.