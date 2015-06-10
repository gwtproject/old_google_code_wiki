# Introduction

DeferredCommand is a very simple utility class for executing code "after the current event has finished executing". It is equivalent to the common Javascript idiom of running code in a window.setTimeout(0). It is used by almost every application either directly or indirectly.

# Details

Very little code is required to implement DeferredCommand, but when we added CommandExecutor, we decided to re-implement DeferredCommand in terms of the more complex class. This made sense from an engineering perspective, but had the undesirable effect of generating much more code for simpler applications. If we were to simply re-implement the simpler form of DeferredCommand, it would benefit smaller apps by reducing their transitive dependencies significantly.