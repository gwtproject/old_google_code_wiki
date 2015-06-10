

# Introduction

This document explains how to run JUnit tests on remote systems.

There are three types of remote RunStyles that can help you run remote tests:
  * Manual
  * Selenium
  * RemoteWeb

To use any of these run styles, you need pass the -runStyle argument into the GWT compiler.  The format looks like this (see specific examples below):
```
-runStyle <NameInCaps>:arguments
```

If you are running a test from Eclipse, you would add something like the following to the VM arguments (Note that the run style name starts with a capital letter):
```
-Dgwt.args="-runStyle Selenium:myhost:4444/*firefox"
```

# Useful Arguments

The following arguments are useful when running remote tests.

## -web

If you are not familiar with Development Mode versus Web Mode, you should read the associated tutorials on that first. All of the following examples assume that you are running tests in Development mode, which requires that you have the [Development Mode Plugin](http://code.google.com/p/google-web-toolkit/source/browse/trunk/plugins/MissingBrowserPlugin.html) installed. Its important to note that URLs must be white listed before the Development Mode plugin will connect to them.  <font color='red'>This means that you must allow the remote connection on the remote system the first time you run the test, or ahead of time if possible.</font>

Tests run in Development Mode by default. You can run a test in web mode by passing the -web argument into GWT.  When running tests in web mode, you do not need to have the Development Mode plugin installed on the remote system, but you cannot debug the test in Eclipse.
```
-Dgwt.args="-web -runStyle Selenium:myhost:4444/*firefox"
```

## -userAgents

When running tests in web mode, GWT compiles the tests for all browsers, which can take a while. If you know which browsers your test will run in, you can limit the browser permutations (and reduce compile time), using the -userAgents argument:
```
-Dgwt.args="-web -userAgents ie6,gecko1_8 -runStyle Selenium:myhost:4444/*firefox"
```


# Run Styles

## Manual

The Manual run style allows you to run JUnit tests in any browser by directing the browser to a URL that GWT provides. For example, if you want to run a test in a single browser, you would use the following arguments.
```
-runStyle Manual:1
```

GWT will now compile your application and test cases, and then show a console message like the following:
```
Please navigate your browser to this URL:
http://172.29.212.75:58339/com.google.gwt.user.User.JUnit/junit.html?gwt.hosted=172.29.212.75:42899
```

Point your browser to the specified URL, and the test will run. You may be prompted by the Development Mode plugin to accept the connection the first time the test is run.

## Selenium

Recommended for Firefox, Safari, Google Chrome, and all browsers other than IE. You can try running IE in Selenium as it is a supported browser. If the tests work for you, then you don't need to use the RemoteWeb runstyle at all, which should simplify your testing.  However, we've found that IE works better with the RemoteWeb run style.

GWT can execute tests against a remote system running the [Selenium Remote Control](http://seleniumhq.org/projects/remote-control/).  You do this using the following command:
```
-Dgwt.args="-runStyle Selenium:myhost:4444/*firefox,myotherhost:4444/*firefox"
```

In the above example, we are using the Selenium run style to execute a development mode test on Firefox against two remote systems (myhost and myotherhost).

### Firefox Profile

By default, Selenium creates a new Firefox profile so it can prevent unnecessary popups that would otherwise mess up the test.  However, you will probably want to create your own Firefox profile that includes the Development Mode plugin.

To do this, run Firefox from the command line and pass in the -ProfileManager argument to open the Profile Manager:
```
firefox.exe -ProfileManager
```

Create a new profile (remember the location), and open it. Setup the profile however you want, making sure to install the Development Mode plugin. On our test systems, we use the following settings:
  * Set a blank homepage
    * Edit->Preferences->Main
    * Set "When Firefox Starts" to "Show a blank page"
  * Disable warnings
    * Edit->Preferences->Security
    * Under "Warning Messages" click "Settings"
    * Uncheck all warnings
  * Disable auto update
    * Edit->Preferences->Advanced->Update
    * Uncheck all automatic updates
  * Disable session restore:
    * Type 'about:config' in the browser bar
    * Find browser.sessionstore.resume\_from\_crash and set it to false
    * Find browser.sessionstore.enabled and set it to false (if it exists)
  * Install [Firebug](http://getfirebug.com/) (useful for debugging)
  * Install GWT Development Mode Plugin
  * Open ports for Plugin.  Since Selenium creates a new profile for each test, you must do this now.  If you do not, you will have to allow the remote connection for every test!
    * Restart Firefox
    * Tools->Addons
    * Select GWT Development Mode Plugin
    * Click "Preferences"
    * Add the IP address that you want to allow the plugin to connect to.

When starting the selenium server, pass in the following argument to use your firefox profile as a template:
```
--firefoxProfileTemplate /path/to/profile
```


## Remote Web

Recommended for IE.

The RemoteWeb run style allows you to run tests against systems running the BrowserManagerServer, a server that GWT provides.

First, you need to start the BrowserManagerServer on the remote test system using the following java command.  Note that gwt-user.jar and gwt-dev.jar are on the classpath.
```
java -cp gwt-user.jar;gwt-dev.jar com.google.gwt.junit.remote.BrowserManagerServer ie8 "C:\Program Files\Internet Explorer\IEXPLORE.EXE"
```

BrowserManagerServer takes commands in pairs.  In the above example, we are associating the name "ie8" with the executable iexplore.exe.
```
<browser name> <path/to/browser>
```

To run a test against IE8, you would use the following argument:
```
-runStyle RemoteWeb:rmi://myhost/ie8
```