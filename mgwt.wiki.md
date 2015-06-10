# mgwt & gwt-phonegap
[mgwt](http://www.m-gwt.com) is a library for developing mobile apps and mobile websites with GWT. mgwt provides native looking widgets for many platforms, animations and many other things that are needed for writing mobile apps. [gwt-phonegap](http://code.google.com/p/gwt-phonegap/) enables GWT apps to use Phonegap. With Phonegap HTML5 apps can access the same device features that native apps can use from Javascript, such as the file system or contacts.
With mgwt and gwt-phonegap you can deploy your GWT apps into any app store or let your users use them as a website.
Both projects are available under apache 2.0 license from maven central.

## Features
mgwt provides mobile widgets that are compatible with UIBinder and the Editor Framework. mgwt integrates touch events and animation events with its own DOM implementation and provides gesture recognizers on top of that.
There are themes for iPhone, iPad, android phones, android tablets and blackberry. For going offline there is an automatically generated HTML5 offline manifest.
In dev mode gwt-phonegap can emulate the Phonegap API so that developing of Phonegap apps can happen inside GWT dev mode. gwt-phonegap also provides utilities to make GWT RPC work in a Phonegap environment.

## Performance
mgwt uses GWT core concepts like Client Bundles, deferred binding, GWT MVP and many more. mgwt tries to leverage GWT features as much as possible, enabling the GWT Compiler to do optimizations.
Mobile devices do have much slower CPUs, less bandwidth and are running on battery. Building inefficient apps means draining the users battery. Since GWT is very good at optimizing Javascript apps it is a natural fit for writing cross platform mobile apps.

mgwt leverages the GWT Compiler and HTML5 to get really good performance on mobile. Here are a few examples:

## No Graphics
Most of the styling can be done only by using CSS3. This reduces apps size significantly. The following screenshots do not contain any images.
<img src='http://misc.mgwt.googlecode.com/git/screenshot/gwt_wiki/slider_iphone.png' width='320px' />
<img src='http://misc.mgwt.googlecode.com/git/screenshot/gwt_wiki/slider_android.png' height='480px' />



## Theming with Client Bundles
mgwt uses GWT Client Bundles for styling to ensure that unused CSS can be removed by the GWT Compiler.  This means e.g. if an app is not using the slider widget the CSS won`t be inside that app.

## Going offline aka no download
Mobile devices are not always connected to the internet. Therefore mobile web apps need to be able to start while being offline. This can be achieved with the HTML5 offline manifest. The GWT compiler sees all artifacts during compilation. This is why mgwt can generate the manifest files for different permutations at compile time. This means that an android device will only download and store the necessary android files, while an iPhone sees a different manifest.

## Minimal markup
Keeping your DOM minimal makes your apps run fast. mgwt tries to use as few DOM elements as possible, while doing styling in CSS.

## Layout with the flexible box model
If layout feels fast the application feels fast. Therefore layout should not be done in Javascript,  CSS should be used instead. The browser can calculate and apply the layout in its native code which is going to be much faster than doing calculations in Javascript.
With HTML5 there is the flexible box model which is supported by all important mobile devices and can be used to achieve any layout that you need for mobile. mgwt uses the flexible box model heavily to build its layouts.

## Animations with CSS3
For mobile applications it is quite common to have animations. If Javascript would be used for animating, the browser has no change to optimize. If you want fast animations you need to give the browser the knowledge of the whole animation, so that it can figure out a fast way to execute it. This can be done with CSS3 animations.
mgwt uses different kinds of CSS animations depending on the platform so that animations always feel fast.



# More
Want to learn more? Checkout the [mgwt homepage](http://www.m-gwt.com) and the [blog](http://blog.daniel-kurka.de/). There is also a 90 minutes [talk](http://www.youtube.com/watch?v=0V0CdhMFiao&feature=plcp) on mgwt from the dutch GTUG:

<a href='http://www.youtube.com/watch?feature=player_embedded&v=0V0CdhMFiao' target='_blank'><img src='http://img.youtube.com/vi/0V0CdhMFiao/0.jpg' width='425' height=344 /></a>



