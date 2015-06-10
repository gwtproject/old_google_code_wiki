# Introduction

The GWT compiler supports basic class metadata through the standard Object.getClass(). This can be useful, but can also lead to a lot of generated strings if used in a context where the compiler cannot determine which classes' metadata is required. It is difficult to avoid this in practice, though, because there are many tricky expressions and dependencies that can cause this "class literal explosion" in practice.

However, it seems to be quite rare for applications to depend upon the specific form of the strings returned from Class.getName(). Thus, if we were to provide a compiler option that would eliminate class literal strings, but preserve their uniqueness, it could make a big difference without breaking applications.

# Performance Impact

  * Significantly less string data reduces overall compilation size.
  * Class literal creation is typically reduced to a no-initializer, `new Class()` operation for non-array, non-enum types.

# Status

Implemented as of r4790 (on Google Code).  The compiler flag `-XdisableClassMetada` must be used to enable this optimization.  Future versions of the compiler may make this the default for obfuscated compilations.  When enabled, the following changes are made to `Class`:
  * `getName` returns a string of the form `Class$<number>`.  The value is stable throughout the lifetime of the application, but will change between runs.
  * `getSuperclass` returns `null`.