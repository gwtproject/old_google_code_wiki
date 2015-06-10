# Introduction

The GWT compiler often emits variable declaration statements of the form `var a,b,c; b=...`. These can be elided slightly into simply `var a, b=..., c`.

Variables and fields can also be initialized slightly more efficiently when they are of the same type. For example:

`a=null; b=0; c=null;` => `a=c=null; b=0;`

# Performance Impact

None.