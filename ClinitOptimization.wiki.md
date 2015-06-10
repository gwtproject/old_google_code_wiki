# Introduction

The GWT compiler currently produces some clinit functions that have no effect other than to ensure that another clinit function is called. For example:

```
function $clinit_1(){
  $clinit_1 = nullMethod;
  $clinit_2();
}
```

In this case, a call to $clinit\_1() could simply be replaced by a call to $clinit\_2(), and the $clinit\_1() itself removed altogether.


# Performance Impact

None, or positive.