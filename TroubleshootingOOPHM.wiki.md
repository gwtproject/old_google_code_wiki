## (Chrome only) Do you see a Gray GWT Toolbox in your omnibar?

![https://google-web-toolkit.googlecode.com/files/chrome-devmode-omnibar-grey-5-2-11.png](https://google-web-toolkit.googlecode.com/files/chrome-devmode-omnibar-grey-5-2-11.png)

The Chrome GWT plugin can't ask for permission like the Safari or Firefox GWT plugins. However, the GWT toolbox will appear in the omnibar whenever you try to use Development Mode. A gray toolbox icon indicates you have a permissions issue. You can click on the gray toolbox to jump to the configuration page and add your host to the whitelist. If your permissions are all in order, the toolbox icon will turn red.

## Are you trying cross-machine debugging?

You need to supply `-bindAddress 0.0.0.0` (or a particular real address) to get the code server to bind on all addresses, rather than just localhost.

## Did you have a compile error?

Take a look at the Development Mode output in the GWT Development Mode window (see <a href='http://www.gwtproject.org/doc/latest/DevGuideCompilingAndDebugging.html#DevGuideDevMode'>using development mode</a>), or if you are using the <a href='https://developers.google.com/eclipse/docs/running_and_debugging_2_0'>Google plugin for Eclipse</a>, check the output in the Development mode panel in your Eclipse IDE to see what happened.

You should see a nice explanation of what to fix.

<br />

## Is Development Mode not working for you?

For all other questions, please visit our <a href='https://groups.google.com/d/forum/google-web-toolkit'>google group</a> for more information.