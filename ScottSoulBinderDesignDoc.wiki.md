# Introduction

Having Scott in the office is useful, but any GWT application would benefit from a little more Scott. Trouble is, he can't be everywhere at once.

![http://google-web-toolkit.googlecode.com/files/ScottSoul.jpg](http://google-web-toolkit.googlecode.com/files/ScottSoul.jpg)

# Goals

  * Improve the usability of GWT
  * Leverage the new HTML5 SoulStorage feature

# Proposal

I propose a new API for accessing Scott's soul. The interface would include useful methods that can access part of Scott's abilities.

```
// TODO(jlabanca): Should this implement HasValue?
interface ScottSoul implements HasValue{

  /**
   * Be sarcastic in an overly complex way.
   *
   * @return true if successful, false if too complicated for anyone to get it.
   */
  boolean complexSarcasm();

  /**
   * Improve the speed of the compiler, optionally introducing
   * an obscure compiler bug that only occurs when passing a
   * null value for an Enum of RPC.
   *
   * @param introduceObscureCompilerBug true to introduce a compiler bug,
   *                                    false to introduce one anyway
   */
  void improveCompilerSpeed(boolean introduceObscureCompilerBug);

  /**
   * Launch a powerful ping pong attack against opponents. Anything higher
   * than level 50 will disarm the opponent of his ping pong paddle. Anything
   * higher than level 75 will disarm the opponent of his life.
   *
   * @param level the attack level - valid values are -1 and 86
   */
  void pingPongAttack(int level);
}
```


This interface can be easily integrated with UiBinder:
```
<ui:UiBinder
  xmlns:ui="urn:ui:com.google.gwt.uibinder">

  <ui:with field="soul" type="com.google.gwt.scott.soul.ScottSoul" />
  
  <g:HTMLPanel>
    <g:Label><ui:msg src="soul.complexSarcasm"/></g:Label>
  <g:/HTMLPanel>

</ui:UiBinder>
```

# Testing

Scott's soul is easily mocked, which makes it great for testing. I'm working on a default implementation that I plan to provide after I resolve an issue with the implementation of complexSarcasm().  For some reason, if you call it too many times, it starts to return false every time.

# Alternatives

We considered the following alternatives, but ultimately chose to go with Scott's soul.

## Tamplin's Soul

Integrating Tamplin's soul would provide more depth, but the API is far too complicated for most users.  It would lead to too much confusion.

## Joel's Soul

Joel's soul is the most cross-browser compatible solution, but its too dynamic to capture in an API. Scott's Soul, on the other hand, is fairly steady.

## LaBanca's Soul

Integrating my own soul was the most obvious choice, but we quickly realized that I have no soul, which wouldn't add much to the library.

```
interface LabancaSoul{
}
```