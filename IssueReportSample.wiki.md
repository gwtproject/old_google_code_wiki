Issue Report Summary:
CloseHandler.onClose() doesn't fire when added to PopupPanel

**Found in GWT Release (e.g. 1.5.3, 1.6 RC)**:

GWT 1.6.2

**Encountered on OS / Browser (e.g. WinXP, IE6-7 | Fedora 10, FF3)**:

WinXP, IE6-8, Chrome, hosted mode | MacOSX Leopard Safari 3.2.1, hosted mode
This is not an issue on WinXP FF3.


**Detailed description (please be as specific as possible)**:

If I add a CloseHandler to a PopupPanel, the onClose() method nevers seems to fire. I've tried adding the same CloseHandler to other widgets like a MenuBar and it worked fine in all browsers there.

You can reproduce this issue by doing the following:

1) Setup your test application with the repro code snippet below.

2) Start the application in hosted mode and click to put focus on the news popup widget.

3) With the news popup open, try closing the popup and expect the onClose() method to fire. You'll notice that on the browsers listed above, the onKeyUp method isn't being fired.

**Shortest code snippet which demonstrates issue (please indicate where actual result differs from expected result)**:

Although in my case I am using a subclass of the PopupPanel (a news panel) to maintain information about the currently selected article, I boiled down the issue to this simple code snippet:

in onModuleLoad() method:

```
final PopupPanel popup = new PopupPanel();
popup.addCloseHandler(new CloseHandler<PopupPanel>() {
  @Override
  public void onClose(CloseEvent<PopupPanel> event) {
    Window.alert("Popup closed");   // <------------ Nothing happens; never see alert dialog box
  }
});
```

**Workaround if you have one**:

Maintain state everytime there is a change to the news panel, but this gets expensive.


**Links to relevant GWT Developer Forum posts**:

http://groups.google.com/group/Google-Web-Toolkit/browse_thread/thread/samplelink