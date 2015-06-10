# Introduction

The Google Web Toolkit (GWT) provides developers with an emulation of a subset of the JRE that includes the java.lang.String class.  In general, a Java String object maps directly to a JavaScript String object.  However, there are a few Java methods that are not provided in JavaScript by default.  For these methods, GWT modifies the JavaScript String prototype.

In a normal compile using the obfuscated style, methods used in the Java source code are obfuscated to a short string in a non-deterministic way.  In the case of these methods, however, the compiler reserves specific two letter names for the GWT compiler named functions, and always maps Java references to these functions in the same way.

The table below shows new functions that the GWT compiler adds to the JavaScript String  prototype.  The name generated depends on the code generation style selected in the compiler:

| String method name | Style OBF | Style PRETTY | Style DETAILED|
|:-------------------|:----------|:-------------|:--------------|
| `getClass`         | `gC`      | `getClass$`    | `getClass__$`      |
| `hashCode`         | `hC`      | `hashCode$`    | `hashCode__$`      |
| `equals`           | `eQ`      | `equals$`      | `equals__Ljava_lang_Object_2$` |
| `toString`         | `tS`      | `toString$`    | `toString__$`      |
| `finalize`         | `fZ`      | `finalize$`    | `finalize__$`      |
| `typeId`           | `tI`      | `typeId$`      | `typeId__$`        |
| `typeName`         | `tN`      | `typeName$`    | `typeName__$`      |
| `charAt`           | `cA`      | `charAt$`      | `charAt__I$`       |
| `compareTo`        | `cT`      | `compareTo$`   | `compareTo__Ljava_lang_Object_2$`|
| `length`           | `lN`      | `length$`      | `length__$`        |
| `subSequence`      | `sS`      | `subSequence$` | `subSequence__II$` |


Authors of hand-written third party script should be aware of these functions and avoid using these names if they wish to add functions to a String object or the String prototype.

# Example

Consider the following code sequence.  The 'noOptimizeFalse()' method is used to obscure the compiler's view of what kind of object 's' really is in order to prevent inlining optimizations for the String class.

```
  String getString() {

    // A little trick to foil compiler inlining.
    Object s = noOptimizeFalse() ? new Object() : "Hello, World!";
    String t = "Goodbye, World!";

    StringBuffer buf = new StringBuffer();
    buf.append("Class is "+s.getClass());
    buf.append("hashCode is "+s.hashCode());
    buf.append("equals is "+s.equals(t));
    buf.append("toString is " +s.toString());

    return buf.toString();
  }

  static native boolean noOptimizeFalse() /*-{
       return false;
  }-*/;

```

When the compiler output style is set to OBF, the compiled output is as follows:

```
zc='Hello, World!';
ed='Goodbye, World!';
function ag(){var a,b,c;b=false?new Cn():zc;c=ed;a=io(new ho());
   jo(a,pd+b.gC());
   jo(a,Ad+b.hC());
   jo(a,n+b.eQ(c));
   jo(a,y+b.tS());
   return lo(a);}
```

Note how the string method names were obfuscated to the names listed in the table.  When the compiler style is set to PRETTY, the compiled output is as follows:

```
function $getString(){
  var buf, s, t;
  s = false?new Object_0():'Hello, World!';
  t = 'Goodbye, World!';
  buf = $StringBuffer(new StringBuffer());
  $append(buf, 'Class is ' + s.getClass$());
  $append(buf, 'hashCode is ' + s.hashCode$());
  $append(buf, 'equals is ' + s.equals$(t));
  $append(buf, 'toString is ' + s.toString$());
  return $toString_0(buf);
}

```

When the compiler is sset to DETAILED, the compiled output is as follows:

```
$intern_6 = 'Hello, World!';
$intern_7 = 'Goodbye, World!';

function com_example_gwt_client_StringPrototypeTest_$getString__Lcom_example_gwt_client_StringPrototypeTest_2(){
  var buf, s, t;
  s = false?new java_lang_Object():$intern_6;
  t = $intern_7;
  buf = java_lang_StringBuffer_$StringBuffer__Ljava_lang_StringBuffer_2(new java_lang_StringBuffer());
  java_lang_StringBuffer_$append__Ljava_lang_StringBuffer_2Ljava_lang_String_2(buf, $intern_8 + s.getClass__$());
  java_lang_StringBuffer_$append__Ljava_lang_StringBuffer_2Ljava_lang_String_2(buf, $intern_9 + s.hashCode__$());
  java_lang_StringBuffer_$append__Ljava_lang_StringBuffer_2Ljava_lang_String_2(buf, $intern_10 + s.equals__Ljava_lang_Object_2$(t));
  java_lang_StringBuffer_$append__Ljava_lang_StringBuffer_2Ljava_lang_String_2(buf, $intern_11 + s.toString__$());
  return java_lang_StringBuffer_$toString__Ljava_lang_StringBuffer_2(buf);
}
```