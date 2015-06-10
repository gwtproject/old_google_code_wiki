# Getting Started with the GWT 2.0 Release Candidates

<i>
Before using GWT 2.0 RC2 or RC1, please understand that they are release candidates and will change before the official release.<br>
We do not recommend using these release candidate builds for production applications.<br>
</i>

This release contains big changes to improve developer productivity, make cross-browser development easier, and produce faster web applications.

## Downloading and Installing GWT 2.0 RC SDKs
No installation process is required to use the GWT SDK; simply unzip it into any convenient directory.

<blockquote>
<a href='http://google-web-toolkit.googlecode.com/files/gwt-2.0.0-rc2.zip'>Download GWT 2.0 RC2</a>
</blockquote>

<blockquote>
<a href='http://google-web-toolkit.googlecode.com/files/gwt-2.0.0-rc1.zip'>Download GWT 2.0 RC1</a>
</blockquote>

The new debugging support in GWT 2.0, called Development Mode, requires a native browser plugin that will be installed on-demand as you use GWT.
See "Getting Started with Development Mode" below for step-by-step instructions for this process.

## Downloading and Installing the Google Plugin for Eclipse 1.2 RCs
Google Plugin for Eclipse 1.2 RC1 includes functionality to support the new features in GWT 2.0, so if you're an Eclipse users, we highly recommend installing it as well.

### Installation Instructions
Download the Google Plugin for Eclipse 1.2.0 Release Candidate zip distributions for your version of Eclipse:

#### RC2
  * 3.5 (Galileo): <a href='http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc2.zip'><a href='http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc2.zip'>http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc2.zip</a></a>
  * 3.4 (Ganymede) <a href='http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc2.zip'><a href='http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc2.zip'>http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc2.zip</a></a>
  * 3.3 (Europa): <a href='http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc2.zip'><a href='http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc2.zip'>http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc2.zip</a></a>

#### RC1
  * 3.5 (Galileo): <a href='http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc1.zip'><a href='http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc1.zip'>http://dl.google.com/eclipse/plugin/3.5/zips/gpe-e35-1.2rc1.zip</a></a>
  * 3.4 (Ganymede) <a href='http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc1.zip'><a href='http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc1.zip'>http://dl.google.com/eclipse/plugin/3.4/zips/gpe-e34-1.2rc1.zip</a></a>
  * 3.3 (Europa): <a href='http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc1.zip'><a href='http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc1.zip'>http://dl.google.com/eclipse/plugin/3.3/zips/gpe-e33-1.2rc1.zip</a></a>

**Note**: Ensure that your version of Eclipse has Eclipse's Web Standard Tools (WST) installed before installing the plugin. WST can be installed by navigating to the Software Installation section, and selecting the the appropriate WST feature from the update site for your version of Eclipse. The update sites and feature names are provided below.

  * 3.5 (Galileo): Galileo -> Web, XML, and Java EE Development -> Eclipse Web Developer Tools
  * 3.4 (Ganymede): Ganymede Update Site -> Web and Java EE Development -> Web Developer Tools
  * 3.3 (Europa): Europa Discovery Site -> Web and JEE Development -> Web Standard Tools Project

It is strongly recommended that you install the release candidate on a fresh install of Eclipse and use a clean workspace.

Finally, follow <a href='http://code.google.com/eclipse/docs/install-from-zip.html'>these instructions</a> for installing the plugin from a zip distribution.

### Important Notes
<dl>
<dt>Terminology changes</dt>
<dd>
We're going to start using the term "development mode" rather than the old term "hosted mode." The term "hosted mode" was sometimes confusing to people, so we'll be using the more descriptive term "development mode" from now on. For similar reasons, we'll be using the term "production mode" rather than "web mode" when referring to compiled script.<br>
</dd>

<dt>Changes to the distribution</dt>
<dd>
Note that there's only one download, and it's no longer platform-specific. You download the same zip file for every development platform. This is made possible by the new plugin approach used to implement development mode (see below). The distribution file does not include the browser plugins themselves; those are downloaded separately the first time you use development mode in a browser that doesn't have the plugin installed.<br>
</dd>
</dl>

### Major New Features in the GWT SDK
<dl>
<dt>In-Browser Development Mode</dt>
<dd>
Prior to 2.0, GWT hosted mode provided a special-purpose "hosted browser" to debug your GWT code. In 2.0, the web page being debugged is viewed within a regular-old browser. Development mode is supported through the use of a native-code plugin called the "Google Web Toolkit Developer Plugin" for many popular browsers. In other words, you can use development mode directly from Safari, Firefox, IE, and Chrome.<br>
</dd>

<dt>Developer-guided Code Splitting</dt>
<dd>
Code splitting using GWT.runAsync(), along with compile reports (also known as <a href='http://code.google.com/events/io/2009/sessions/StoryCompilerGwtCompiler.html'>The Story of Your Compile</a>) allows you to chunk your GWT code into multiple fragments for faster startup. Imagine having to download a whole movie before being able to watch it. Well, that's what you have to do with most Ajax apps these days -- download the whole thing before using it. With code splitting, you can arrange to load just the minimum script needed to get the application running and the user interacting, while the rest of the app is downloaded as needed.<br>
</dd>

<dt>Declarative User Interfaces with UiBinder</dt>
<dd>
GWT's <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/uibinder/client/UiBinder.html'>UiBinder</a> now allows you to create  user interfaces mostly declaratively. Previously, widgets had to be created and assembled programmatically, requiring lots of code. Now, you can use XML to declare your UI, making the code more readable, easier to maintain, and faster to develop. The Mail sample has been updated to show a practical example of using UiBinder.<br>
</dd>

<dt>New Layout Panels for Fast and Predictable Layout</dt>
<dd>
A number of new panels were added, which together form a stable basis for fast and predictable application-level layout. The official doc is still in progress, but for an overview please see <a href='http://code.google.com/p/google-web-toolkit/wiki/LayoutDesign'>Layout Design</a> on the wiki. The new set of panels includes <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/RootLayoutPanel.html'>RootLayoutPanel</a>, <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/LayoutPanel.html'>LayoutPanel</a>, <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/DockLayoutPanel.html'>DockLayoutPanel</a>, <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/SplitLayoutPanel.html'>SplitLayoutPanel</a>, <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/StackLayoutPanel.html'>StackLayoutPanel</a>, and <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/TabLayoutPanel.html'>TabLayoutPanel</a>.<br>
</dd>

<dt>Bundling of Resources via <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/resources/client/ClientBundle.html'>ClientBundle</a></dt>
<dd>
GWT introduced <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/user/client/ui/ImageBundle.html'>ImageBundle</a> in 1.4 to provide automatic spriting of images. ClientBundle generalizes this technique, bringing the power of combining and optimizing resources into one download to things like text files, CSS, and XML. This means fewer network round trips, which in turn can decrease application latency -- especially on mobile applications.<br>
</dd>

<dt>Simplified Unit Testing with HtmlUnit</dt>
<dd>
Using HtmlUnit for running test cases based on <a href='http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/junit/client/GWTTestCase.html'>GWTTestCase</a>: Prior to 2.0, GWTTestCase relied on SWT and native code versions of actual browsers to run unit tests. As a result, running unit tests required starting an actual browser. As of 2.0, GWTTestCase no longer uses SWT or native code. Instead, it uses HtmlUnit as the built-in browser. Because HtmlUnit is written entirely in the Java language, there is no longer any native code involved in typical test-driven development. Debugging GWT Tests in development mode can be done entirely in a Java debugger.<br>
</dd>
</dl>

### Major New Features in the Google Plugin for Eclipse
<dl>
<dt>Development Mode Launch View</dt>
<dd>
Integrates your Development Mode logs right into Eclipse, which means one less window to shuffle around<br>
</dd>

<dt>UiBinder Support</dt>
<dd>
The UiBinder template editor provides auto-completion and formatting for editing ui.xml files (and embedded CSS blocks)<br />
Validation of UiBinder templates and backing Java classes<br />
New UiBinder wizard to quickly get started<br>
</dd>

<dt>ClientBundle Support</dt>
<dd>
"New ClientBundle" wizard to bundle CSS and other resources together to minimize HTTP round-trips<br />
As-you-type validation ensures that your app's static resources are always in the right place<br>
</dd>

<dt>RPC Refactoring</dt>
<dd>
Automatically updates sync/async pairs of RPC interfaces and their methods<br>
</dd>

<dt>JNSI Reference Auto-completion</dt>
<dd>
Auto-completion takes the pain out of referencing Java members from JSNI methods<br>
</dd>
</dl>

### Getting Started with Development Mode
#### Step 1: Start Development Mode
The first time you launch development mode in GWT 2.0, you might be surprised that not much seems to happen.
All you will see is something like this on the console:
<blockquote>
<pre>
Using a browser with the GWT Developer Plugin, please browse to<br>
the following URL:<br>
http://192.168.1.42:8888/Hello.html?gwt.codesvr=192.168.1.42:9997<br>
</pre>
(where <code>192.168.1.42</code> is your own IP address)<br>
</blockquote>
That's because you're now in control over which target browser launches and when.

#### Step 2: Open a browser
Open the browser you want to use to view the GWT application you're debugging.

If this is the first time you've ever attempt to use development mode with that browser, you will not yet have the GWT Developer Plugin installed, so you'll see a page that instructs you to install the plugin. Once you download and install the plugin, restart the browser and navigate again to the URL from above, making sure it includes the `gwt.codesvr` query parameter.

<blockquote><i>
When installing the GWT Developer Plugin on Windows 7, certain configurations may require you to explicitly save the installer as a file and then "Run as Administrator".<br>
</i></blockquote>

Note that the first time you access your GWT module from a browser, it may appear to hang momentarily as your module loads. This is normal. Subsequent refreshes within the browser are much faster as long as you keep the development mode session running.

#### Step 3: Follow the Edit/Save/Refresh cycle without restarting development mode
At this point, use your Java debugger as normal.

Because refeshes are much faster than the initial load, it is strongly recommended that you simply leave development mode running, because you can continue to edit and save any of your projects files -- including `.html`, `.css`, `.java`, `.gwt.xml`, and `.ui.xml` files -- and simply refresh to pick up changes.

#### Step 4: Open additional browsers for testing
You can open any number of additional browsers to ensure your application behaves properly across browsers.

### Breaking changes and known issues/bugs/problems
  * Windows users who have previously installed the Google Web Toolkit Developer Plugin for IE will have to uninstall the old version. Use the following steps:
    1. From the Windows "Start" Menu, open "Control Panel"
    1. Select "Add/Remove Programs"
    1. Select "Google Web Toolkit Developer Plugin for IE" then click "Uninstall"
    1. Run Internet Explorer and browse to <a href='http://gwt.google.com/samples/MissingPlugin'><a href='http://gwt.google.com/samples/MissingPlugin'>http://gwt.google.com/samples/MissingPlugin</a></a> to install the new version of the plugin.
  * The -portHosted Development Mode flag has been renamed to -codeServerPort.
  * The release notes included with RC2 contain one inaccuracy: the Image.onload event still does not fire on Internet Explorer when image is in cache.  See issue <a href='http://code.google.com/p/google-web-toolkit/issues/detail?can=2&q=863'>863</a> for more details.
  * Prior to 2.0, GWT tools such as the compiler were provide in a platform-specific jar (that is, with names like `gwt-dev-windows.jar`). As of 2.0, GWT tools are no longer platform specific and they reside in generically-named `gwt-dev.jar`. You are quite likely to have to update build scripts to remove the platform-specific suffix, but that's the extent of it.
  * The development mode entry point has changed a few times since GWT 1.0. It was originally called `GWTShell`, and in GWT 1.6 a replacement entry point called `HostedMode` was introduced. As of GWT 2.0, to reflect the new "development mode" terminology, the new entry point for development mode is `com.google.gwt.dev.DevMode`. Sorry to keep changing that on ya, but the good news is that the prior entry point still works. But, to really stay current, we recommend you switch to the new `DevMode` entry point.
  * Also due to the "development mode" terminology change, the name of the ant build target produced by `webAppCreator` has changed from `hosted` to `devmode`. In other words, to start development mode from the command-line, type `ant devmode`.
  * HtmlUnit does not attempt to emulate authentic browser layout. Consequently, tests that are sensitive to browser layout are very likely to fail. However, since GWTTestCase supports other methods of running tests, such as Selenium, that do support accurate layout testing, it can still make sense to keep layout-sensitive tests in the same test case as non-layout-sensitive tests. If you want such tests to be ignored by HtmlUnit, simply annotate the test methods with @DoNotRunWith({Platform.HtmlUnit}).
  * Versions of Google Plugin for Eclipse prior to 1.2 will only allow you to add GWT release directories that include a file with a name like `gwt-dev-windows.jar`. You can fool it by sym linking or copying gwt-dev.jar to the appropriate name.
  * The way arguments are passed to the GWT testing infrastructure has been revamped. There is now a consistent syntax to support arbitrary "run styles", including user-written, with no changes to GWT itself. For example, `-selenium FF3` has become `-runStyle selenium:FF3`. This change likely does not affect typical test invocation scripts, but if you do use `-Dgwt.args` to pass arguments to GWTTestCase, be aware that you may need to make some changes.
  * When using ClientBundle, be aware that images using alpha transparency do not appear transparent in IE6. The Mail sample application included in the GWT distribution currently suffers from this limitation (that is, the images have opaque backgrounds when viewed on IE6).