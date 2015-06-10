# Design: Out of Process Hosted Mode (OOPHM)

## Introduction & Motivation
GWT's hosted mode browser is an essential part of developing GWT applications. It allows developers to use a standard java debugger to debug GWT/Java code while that code actually affects a real production browser. The current architecture leverages the SWT browser bindings to run the browser instance inside of the hosted mode process. This approach has proved limiting for a number of reasons.

## Limitations of the current approach
  * It is difficult to support new versions of browsers. For example, we still use Mozilla 1.7.12 on Linux and a custom WebKit build on Mac OS X.
  * Due to the way SWT embeds the browser, many plugins/extensions do not work. (Firebug, Google Gears, DOM Inspector)
  * There are unpleasant AWT/SWT interactions that continue to require attention. Also, our reliance on AWT has increased in the past few releases and this is expected to continue.
  * We only support one browser per platform (theoretically this could be worked around, but it would require a lot of work and have very high maintenance cost.
  * We can't use hosted mode across platforms (for example, using IE from a Linux hosted mode across the network). Fixing a late bug on IE often requires setting up an IDE and importing the entire project.

## Goals
  * Support use of multiple browsers on each supported platform: Linux: Firefox 1.5+. Windows: Chrome, Firefox 3+, IE6/7/8 and Safari3+. OS X: Firefox 1.5+ and any WebKit browser.
  * Enable the use of standard and current browser plugins, tools and capabilities (Firebug, DOM Inspector, Gears).
  * Avoid version dependencies in supported browsers or system-supplied libraries (and minimize it where it is absolutely not possible).
  * Provide user-visible performance no worse than the current implementation.
  * Do nothing to impede "instant hosted mode" plans.
  * User should be able to start a hosted mode session directly from the IDE, as it currently possible. This includes being able to debug that process in a meaningful way.
  * Continue to support -noserver functionality and use cases.
  * Minimize the total number of plugins required (i.e. favor cross-browser plugins over their browser specific brethren).
  * Minimize platform-specific code. We should no longer need a gwt-dev-xxx.jar.

## Non-goals
  * We are specifically not trying to implement "instant hosted mode" with this change, although we don't want to do anything to prevent it later.
  * Hosted mode across a high-latency network will not be specifically supported, but may work with limitations.
  * Opera support. (This could become a goal at a later date)
  * Provide an interface for third-party tools to leverage our communication protocols to the browser. (This could become a goal at a later date)

## Use Cases
  1. Retain the original use case - A GWT developer should be able to launch and debug a GWT application from within a standard Java debugger and IDE. This means that the spawning process must be a jvm instance and we must not do anything to obscure useful stack trace information.
  1. Debugging in multiple browsers - A GWT developer should be able to launch and debug the same GWT application (or different applications) from different browser instances. Of course, the browsers can be the same type of browser or multiple tabs in the same browser. (There is, however, one caveat in debugging two applications in different tabs. Most browsers have a single event queue for all of the tabs. So a breakpoint in one of the applications will prevent a tab switch so long as execution is suspended.)
  1. Remote debugging - I'm not including this on at this point.

## Design & Architecture
> ### Overview

> The diagram below gives a high-level picture of how all the parts fit together in out-of-process mode. Each of the different components shown is explained in greater detail in the following sections.

> ![http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-arch.png](http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-arch.png)

> ### User Interface
> The following is an **incomplete** sketch of the new UI for hosted mode. The mocks will be updated again soon, but this illustrates the primary motivation for updating the UI. The possibility of having multiple browsers running modules in the same hosted mode server requires more visibility and separation of the different clients and their associated reporting. Another component that is not represented currently is the embedded Tomcat instance. That will be fixed in the next mock.  _(Note: this mock is rather out-of-date at this point)._



> ![http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-ui.png](http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-ui.png)

> ### Browser Channel / Communications Protocol
> All communication between the hosted GWT module and the corresponding JavaScript environment (browser) takes place via a TCP socket. The two sides communicate through asynchronous message passing to allow method invocations to be re-entrant onto the same thread. This maintains the constraint that the hosted mode process be debuggable with a standard Java debugger. A simple example of a re-entrant invocation is given below which demonstrates the need for non-synchronous dispatch. A channel is established for each GWT module that is being hosted and the channel setup is initiate from the Browser Plugin. The hosted mode process acts as a TCP server listening for connections and instantiating modules (and their associated infrastructure) on demand. Note this also means that multiple modules on a single host page will establish multiple channels.

> Consider the following GWT code:

```
  public class MyEntryPoint implements EntryPoint {
    private static native int jsniMethod() /*-{
      return 1;
    }-*/;

    public void onModuleLoad() {
      jsniMethod();
    }
  }
```

> Executing this code in the hosted mode browser requires the following steps:
  1. **JavaScript:** the browser plugin sends a `LoadModuleMessage` with the module name.
  1. **Java:** the hosted mode server receives the `LoadModuleMessage`, loads the module and invokes the `onModuleLoad` in the corresponding EntryPoints. In this case `MyEntryPoint::onModuleLoad` is called. When `MyEntryPoint` is compiled, a `LoadJsniMessage` is sent to create browser-side JavaScript functions for each JSNI method, then when `onModuleLoad` invokes `jsniMethod` an `InvokeMessage` is sent.
  1. **JavaScript:** This is the key part of the example. The JavaScript engine is currently awaiting a return from the `LoadModuleMessage` it sent, but it must be in a position to invoke the call to `MyEntryPoint::jsniMethod` on the same thread. This is accomplished by having the thread enter a read-and-dispatch routine following every remote invocation. In this case, the thread receives the `LoadJsniMessage` and `InvokeMessage` messages, invokes `jsniMethod` and sends a `ReturnMessage` containing the value 1.
  1. **Java:** The read-and-dispatch routine receives the `ReturnMessage` and knows to return from the call to `jsniMethod`. Having fully executed the `onModuleLoad` method it sends a `ReturnMessage` and falls back into a top level read-and-dispatch loop. (Since all calls originate from the browser's UI event dispatch, only the hosted mode server needs to remain in a read-and-dispatch routine during idle time. The browser simply returns control by exiting the JavaScript function that was originally called.)

> To further illustrate this functionality, the following is a simplified state diagram shows how the messaging scheme simulates method invocation over an asynchronous messaging channel.

> ![http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-state.png](http://google-web-toolkit.googlecode.com/svn/wiki/DesignOOPHM-state.png)

> The wire format for the communications protocol is a simple binary format. A need may arise for something more elaborate at a later date, but we have elected for the simplest possible scheme that works for now. The details of each message's binary format is given below along with formats for primitive data types.

> #### Messages
> NOTE: This is likely not a complete list of the messages that exist in the system, and values given for some fields are likely to change.

> _LoadModuleMessage:_ requests that the hosted mode server load and begin executing a module.
> | type (byte) =2 | version (int) =1 | module name (string) | user agent (string) |
|:---------------|:-----------------|:---------------------|:--------------------|

> _InvokeMessage:_ used to do method invocation on Java and JavaScript objects. This message's format is asymmetric.
> From server to client (invoke a method on a JavaScript object):
> | type (byte) =0 | method name (string) | this (Value) | number of args (int) | args (Value[.md](.md)) |
|:---------------|:---------------------|:-------------|:---------------------|:-----------------------|
> From client to server (invoke a method on a Java object):
> | type (byte) =0 | method dispatch id (int) | this (Value) | number of args (int) | args (Value[.md](.md)) |

> _LoadJsniMessage:_ used to evaluate JavaScript code in the browser to initialize JSNI methods.
> | type (byte) =4 | js code (string) |
|:---------------|:-----------------|

> _QuitMessage:_ used to cooperatively shutdown the browser channel.
> | type (byte) =3 |
|:---------------|

> _ReturnMessage:_ - used to send the return values associated with Invoke, InvokeSpecial and LoadModule messages.
> | type (byte) =1 | is exception (boolean) | return value (Value) |
|:---------------|:-----------------------|:---------------------|

> _InvokeSpecialMessage:_ - used to access Java object properties (fields or method _pointers_) from the browser side.
> | type (byte) =5 | special method (byte) | number of args (int) | args (Value[.md](.md)) |
|:---------------|:----------------------|:---------------------|:-----------------------|

> _FreeValueMessage:_ - used to tell the other side it can free its Java (server-side) or JavaScript (browser-side) objects
> | type (byte) =6 | number of ref ids (int) | ref ids (int[.md](.md)) |
|:---------------|:------------------------|:------------------------|

> `*` all strings are encoded as a length, n, followed by n bytes of data containing the string in utf8 encoding.

> `*``*` the encoding of values is given below.

> #### Values
> NOTE: the _tag_ part is only needed for generic `Value`s passed in Invoke, InvokeSpecial and Return messages.

> _null_
> | tag (byte) =0 |
|:--------------|

> _undefined_ (also used for `void` returns)
> | tag (byte) =12 |
|:---------------|

> _boolean_
> | tag (byte) =1 | value (8 bit signed) |
|:--------------|:---------------------|

> _byte_
> | tag (byte) =2 | value (8 bit signed) |
|:--------------|:---------------------|

> _char_
> | tag (byte) =3 | value (16 bit signed) |
|:--------------|:----------------------|

> _short_
> | tag (byte) =4 | value (16 bit signed) |
|:--------------|:----------------------|

> _int_
> | tag (byte) =5 | value (32 bit signed) |
|:--------------|:----------------------|

> _long_ (unused!)
> | tag (byte) =6 | value (64 bit signed) |
|:--------------|:----------------------|

> _float_
> | tag (byte) =7 | value (32 bit IEEE 754 ) |
|:--------------|:-------------------------|

> _double_
> | tag (byte) =8 | value (64 bit IEEE 754 ) |
|:--------------|:-------------------------|

> _string_
> | tag (byte) =9 | length (32 bit signed) | data (utf8 data, variable length) |
|:--------------|:-----------------------|:----------------------------------|

> _java object (this is an instance that exists in the JVM process)_
> | tag (byte) =10 | ref id (32 bit signed) |
|:---------------|:-----------------------|

> _javascript object (this is an instance that exists in the browser process)_
> | tag (byte) =11 | ref id (32 bit signed) |
|:---------------|:-----------------------|

> ### Browser Plugin
> The browser plugin is responsible for handling and dispatch messages in the browser and also for interacting with the browser's JavaScript engine. Each plugin consists of two conceptual parts: browser-specific functionality for interacting with the JavaScript engine and a set of shared C++ classes that implement the communication channel and message serialization. We continue to make every effort to implement the plugins using common and standard APIs (like NPAPI/npruntime), but where that is insufficient we rely on proprietary (but public) plugin APIs. Below is a list of the popular browsers and the APIs we are using, or planning to use.

> WebKit - WebKit (WBPL) Plugin _(Sadly, npruntime has limitations we haven't been able to overcome in WebKit)_

> Mozilla - XPCOM component _(NPAPI was initially used and it is mostly functional, but we ran into a problem with Window.enableScrolling that was insurmountable)_

> IE6/7/8 - ActiveX control

> Opera - Unsupported (NPAPI/npruntime when we confirm that they have finally implemented it fully)

> Chrome - Preliminary support via CRX with NPAPI plugin

> ### Hosted GWT module space
> The infrastructure in place in the current version of hosted mode has remained largely intact. At a very high level, this new model for hosted mode replaces the implementation of the JavaScriptHost interface (which provides an interface directly to the corresponding JavaScript environment) with the BrowserChannel construct that is described above. We are intentionally avoiding a massive restructuring of the hosted space infrastructure at this point.

# Security Considerations

## Primary Threats
  * The plugin is available to arbitrary and potentially malicious JS code.  If not protected, this could lead to buffer overrun and similar attacks.
  * The plugin opens a network connection to an address supplied by JS code.  This could be used to subvert a firewall for example.  It is clear the plugin could be used to do at least a port scan behind a firewall, and it may be possible to do more.
  * The plugin connects to a GWT server, which could be under hostile control.  Lack of care in processing this connection could result in buffer overrun and similar attacks.

## Threat Mitigation
  * All data passed from JS to the plugin must be fully validated.  We do not assume any subversion of the host browser, as in that case the presence of our plugin does not increase the exposure.
  * The plugin maintains a whitelist (location varies by browser) listing the hosts which are allowed to open an OOPHM network connection, initially containing only localhost and 127.0.01.  Any URL whose serving host is not in the whitelist will result in a dialog box asking the user to permit the connection (with the option of remembering this decision for future connections).  If the URL is blacklisted or the user does not give consent, the connection attempt fails.
    * The normal use case is for a developer debugging their own GWT apps, usually on the same machine but sometimes with other nearby machines, so it should be surprising for a developer to browse a site not under their control and see a dialog about the connection attempt.
    * Note that this consent is essentially at the host level, so if a user trusts a host they are trusting all the web pages served from that host.
  * Rogue GWT servers should not be an issue due to the whitelist approach described above, but we still make sure not to overflow any buffers due to malformed network messages.  We do not attempt to avoid denial of service attacks, such as exhausting all memory to hold a long string, etc.