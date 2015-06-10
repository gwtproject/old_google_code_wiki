# Introduction

This is a detailed description of how to run OOPHM if you're a Mac-based eclipse-toting GWT contributor. If you don't know an OOPHM is, you should first read: [UsingOOPHM](UsingOOPHM.md)

# Details

Suppose you work with GWT from source, you've imported the Hello project, and you want to run it under OOPHM. You must:

  * Make sure your oophm jar is up to date
    * cd trunk
    * ant clean
    * ant dist-dev
  * Back in eclipse, go to the Package Explorer, right click gwt-dev-mac and choose Properties
  * Java Build Path / Libraries
  * Delete the GWT\_TOOLS/lib/eclipse/org.eclipse.swt.carbon-macosx... jar
  * Click OK
  * Now choose Run -> Run Configurations
  * Choose Hello and click the Classpath tab
  * Select User Entries
  * Add External JARS...
  * select trunk/build/lib/gwt-dev-oophm.jar
  * Move the oophm jar above Hello (default classpath)
  * Check the Arguments tab and ensure that -XstartOnFirstThread is not there (it probably isn't, and you don't need it in any case)
  * Rebuild and run, ignoring all the errors in gwt-dev-mac'
  * Don't fret when you see the IOException in the stylish OOPHM window, it's just failing to exec Firefox.
  * Open Safari and point it at the URL you see in the OOPHM log window, in this case something like http://localhost:8888/Hello.html?gwt.hosted=127.0.0.1:9997

To run a GWTTestCase in Eclipse, you'll have to undo all this.

  * Go to the Package Explorer, right click gwt-dev-mac and choose Properties
  * Java Build Path / Libraries
  * Add Variable..
  * Select GWT\_TOOLS and click Extend...
  * Open lib/eclipse and pick org.eclipse.swt.carbon-macosx...jar
  * Delete the GWT\_TOOLS/lib/eclipse/org.eclipse.swt.carbon-macosx... jar
  * OK, OK, OK
  * Now choose Run -> Run Configurations
  * Choose Hello and click the Classpath tab
  * Select User Entries
  * Remove the oophm jar