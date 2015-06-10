#Describes the when-linker-added tag

# Introduction

There are a couple of different ways that the choice of linker affects
what the compiler should do with runAsync:
  * If the code will be loaded inside a nested function, then variable  and function definitions need to be rewritten if they are referenced on a different fragment than the one that loads them.
  * If the code could be loaded in violation of the same origin policy, then a script tag rather than XHR needs to be used to download the code.

The when-linker-added tag is intended to allow various aspects of compilation to vary depending on whether a linker is present.  To solve the rewrites, have a configuration property that is flipped on whenever the XS linker is included.  To use a script tag rather than XHR to download deferred code, use a deferred binding that fires only when the XS linker is included.

# Details

The when-linker-added tag acts like any other condition that can be tested in a module file.  For example:
```
  <replace-with 
    class="com.google.gwt.core.client.impl.CrossSiteLoadingStrategy"> 
    <when-type-is 
class="com.google.gwt.core.client.impl.AsyncFragmentLoader.LoadingStrategy" /> 
    <when-linkers-include name="xs" /> 
  </replace-with> 
```

The condition takes effect whenever the linker with the given name is active for the present module.


# Links

The initial discussion is here:

http://groups.google.com/group/google-web-toolkit-contributors/browse_thread/thread/ebca7dc3b6171ea9/0b7c699b5775c046?lnk=raot

An intermediate patch is here:

http://gwt-code-reviews.appspot.com/143811


Discussion about that intermediate patch is here:

https://groups.google.com/group/google-web-toolkit-contributors/browse_thread/thread/bd7936cd2f323484/87009f6a5c5b6606?lnk=raot&pli=1

The final patch committed is here:

http://gwt-code-reviews.appspot.com/150801