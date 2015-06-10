# Introduction

The JRE HashMap (and HashSet) class is a fundamental building-block of many applications. Unfortunately its semantics are significantly more complex than the built-in Javascript object's string->value map. The GWT HashMap implementation is thus rather involved, leading to code bloat (especially in smaller applications where a single large class is quite noticeable).

# Details

There are several reasons that the underlying Javascript object type is insufficient for implementing HashMap.
  * It's just a String->value map, not an Object->Object map.
  * There is no provision for overriding the equality behavior for hash keys (unlike Java).
  * There are also certain key values that are off-limits in Javascript objects, making them insufficient even for simple String->Object maps.

To make matters worse, the inability to use objects as keys makes it impossible to create a Set type (unless you just want a set of Strings).

# Options

For simple String->Object maps, the simplest option is to create a simple map implementation on top of a Javascript object. This can be accomplished easily using JSNI (though you still need to prefix your keys with a special character to avoid stepping on certain Javascript keywords), but is insufficient for many use-cases.

The second option is to optimize GWT's JRE HashMap implementation. The current implementation makes heavy use of other collections, sets, and utility classes, so it seems likely that a careful optimization of what's there with an eye towards eliminating dependencies could yield a much smaller implementation.

Finally, while it would be preferable to optimize the JRE HashMap, it might also be possible to create a replacement Map type with simpler semantics (perhaps something simple with only get(), put(), and iterator() methods).