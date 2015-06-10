# Introduction

Candidates:
  * `true`, `false`, `null`
    * `var _=null, t=true, f=false;`
  * `prototype`
    * `var p='prototype'; Class[p] = new Super(); `
  * `throw`
    * `function t(e) { throw e; } t(new Exception())`

# Performance Impact

These expressions are slower to execute on almost all browsers, up to 20%. Some of them, such as throw, are executed so infrequently that they are probably still worth it. The rest might be appropriate only when code size is of the utmost importance.

Interning `null` might be irrelevant once we take into account these [null optimizations](NullOptimization.md), because a large percentage of them will go away anyway.