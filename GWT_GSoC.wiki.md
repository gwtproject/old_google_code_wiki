#summary Google Web Toolkit (GWT) Ideas List for Google Summer of Code 2007
# Google Web Toolkit (GWT) Ideas List for Google Summer of Code 2007
This page lists several ideas for student projects related to the Google Web Toolkit (GWT), as part of the [Google Summer of Code](http://code.google.com/soc). If you are unfamiliar with GWT, please visit the [Google Web Toolkit website](http://code.google.com/webtoolkit/).

If you would like to participate in any of these projects, please visit the [SoC Student Application Page](http://groups.google.com/group/google-summer-of-code-announce/web/guide-to-the-gsoc-web-app-for-student-applicants)!

If you are curious to know more about the philosophy and development practices of the Google Web Toolkit project, please visit our guide on [Making GWT Better](http://code.google.com/webtoolkit/makinggwtbetter.html).

Please note: we have numbered the projects below, mostly to allow for easy reference, but also loosely based on priority to GWT's current roadmap.

### 1. Java API Compatibility Validator
As a software API evolves, it may undergo refactorings.  Unit tests can help determine when behavior changes, but it is difficult to test every possible use case of a given method, class, or interface.  A tool that could examine two versions of a set of Java classes and report on interface changes that affect user code would be extremely helpful in guiding API changes.

A thumbnail sketch of such a tool might be:
  * Scans a Java package, discovering classes and methods via reflection, and recording the public interfaces
  * Compares the "footprints" of two packages, identifying differences, such as:
    * Added or deleted methods/fields
    * New/missing interfaces
    * Visibility changes on members (such as from going from public to protected)
    * etc.

**Difficulty Level**: Medium/Hard

**Skills Required**:
  * Good familiarity with the Java reflection API
  * Good understanding of the Java language in general


### 2. GWT for Gadgets
GWT can currently be used to develop "gadgets" or "widgets" (such as Google Gadgets and Mac OS X Widgets), but only with difficulty.  Improvements to GWT to let it be used more easily in this area would be very helpful. There might be some overlap or shared work between this project and the "Googel for Firefox Extensions" project.

**Difficulty Level**: Easy/Medium

**Skills Required**:
  * Experience with Java and JavaScript
  * Experience with Google Gadgets and Mac OS X Widgets (or a willingness to learn them)
  * A user's knowledge of GWT


### 3. Vector Graphics API for GWT
A vector API would be useful to developers interested in using GWT to build rich, highly interactive user interfaces similar to Google Maps or Flash applets.  Currently, the ecosystem of vector graphics engines across browsers and operating systems is highly diverse.  Just as essential as implementing the library is the research required to establish a solid cross-platform strategy.  There would be three phases to this project:
  * Researching each vector graphics engine (Flash, SVG, WPF, Canvas) to determine which engine/OS combinations support which functionality
  * Using the results of the research, work with the GWT team to determine a final list of features that can be supported cross-platform (for example, is it possible to reliably support mouse events in any browser/OS?)
  * Work with the GWT team to design a final Vector API, and then implement that API

**Difficulty Level**:  Medium

**Skills Required**:
  * Basic experience with (and access to) Windows, Linux, and Mac OS X
  * Basic JavaScript experience
  * Basic understanding of Vector Graphics and UI implementations
  * Experience with Java


### 4. Network Sockets API for GWT
A frequent hurdle encountered by developers using GWT is a lack of robust network services.  Flash provides networking functionality that JavaScript does not natively support, but using this functionality from within a GWT application is not easy or intuitive.  An API and library that abstracts the loading of a small Flash applet and corresponding communications, exporting a clean interface into GWT user code would be helpful.

**Difficulty Level**: Easy/Medium

**Skills Required**:
  * Ability (and required software) to create a simple Flash applet
  * Experience with Java
  * Experience with JavaScript


### 5. GWT for Firefox Extensions
The Firefox extension model can be grossly summarized as a JavaScript application that operates over an XML/XUL DOM instead of an HTML DOM.  It would be extremely cool if GWT could be used to develop Firefox extensions. Such a project would consist of two parts, which are semi-independent projects:
  * Port the GWT UI library to use a XUL DOM (instead of an HTML DOM)
  * Create a new GWT bootstrapping mode that's compatible with Firefox extensions
Some coordination would be required between the two sub-projects, to agree on common infrastructure.

**Difficulty Level**: Medium (UI changes) or Hard (bootstrapping changes)

**Skills Required**:
  * Experience with Java
  * Experience with JavaScript
  * Experience with Firefox extensions (or a willingness to learn them)
  * Experience with XML
  * Experience with GWT's UI library would be helpful


### 6. GWT for Cell Phones
Developing AJAX applications for cell phones is challenging, especially if the developer wants to reuse the same source code for both cell phones and desktop browsers.  If GWT could offer developers a path to deploy their GWT applications to cell phones with little or minimal re-coding, it would be a major benefit to GWT and the web in general.  This project would have several sub-tasks:
  * An analysis of what behaviors need to be changed for cell phones
    * For example, Labels would probably work identically on desktops and phones, but HorizontalPanel might need to have different behavior on a cell phone than on a desktop
  * A survey of available cell phone browsers, and an analysis of their capabilities (to determine which can and which can't be supported)
  * Updates to the GWT UI library, including implementations of the Widgets for selected cell phone browsers

**Difficulty Level**: Hard

**Skills Required**:
  * Experience with (and access to) mobile phone browsers, and their behaviors (such as how Opera Mobile modifies its layout algorithm based on the orientation of the phone)
  * Experience with JavaScript and Java
  * Solid understanding of HTML DOMs and how rendering occurs in the browser



