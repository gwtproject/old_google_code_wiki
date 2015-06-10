# Introduction

Previously, we have embedded the browser inside GWT hosted mode using SWT.  This is unsatisfactory for a number of reasons:
  * SWT has native pointers, so you have to build it for 32 or 64 bits (and we currently only ship 32-bit binaries, so you can't use it with a 64-bit JVM).
  * You can only use one browser per platform in hosted mode.
  * You can't easily use tools like DOM Inspector, Firebug, etc.
  * On Linux, we have to ship an embeddable version of Mozilla 1.7.12, which is a very old browser.  Also, distributing a large binary like this is problematic to support on a wide range of distributions due to shared library dependencies.

The solution is to invert the problem -- instead of embedding the browser in hosted mode, we will embed a hosted mode plugin in the browser.  This has a much smaller footprint that is easier to support, and then we can get support for multiple browsers per platform and cross-machine hosted mode (ie, running hosted mode on Linux and connecting to it from IE on a Windows machine).

# Installing the Plugin

You will need to install a plugin in each browser you intend to use with OOPHM.

If you start DevMode in a browser without the plugin, you will get to the page allowing you to install the plugin.  If you want to install it ahead of time, you can go directly
to that [missing-plugin page](http://gwt.google.com/samples/MissingPlugin/) to install the plugin.

The plugins currently available for the following browsers/platforms:
  * **Google Chrome (Windows x86 only, Dev channel)**
  * **Firefox on Windows (x86), Mac (PPC/x86), or Linux (x86/x86\_64)**
  * **Safari 3/4 on MacOSX (PPC/x86 - not yet on Snow Leopard)**
  * **IE6/7/8 (32-bit IE only)**

# Using OOPHM
OOPHM is currently in trunk (it is not and will not be available with any version of GWT earlier than 2.0), and is now enabled by default.

If you are using the Eclipse plugin, you may need to manually create a plain Java launch config until version 1.2 is available (1.12 has some support).

Note that calls between Java and JS are synchronous, and that means the plugin has to block the browser while a Java method is executing.  If you are debugging your Java code, the browser will appear hung until you return back to browser-side code.