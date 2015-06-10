# Details of ARIA Implementation in GWT

Accessibility (a11y) support was added to GWT in version 1.5. Support comes in the form of:

  * Adding default tabindex values to make naturally focusable DOM elements receive keyboard events by default
  * Explicitly removing IFRAMES used for history support and boostrapping from the tab cycle
  * Adding keyboard support to GWT widgets
  * Instrumenting GWT widgets which do not derive from native HTML controls with ARIA roles and states

While the first three items on the list apply to any platform, the ARIA specification is new, so not all Assistive Technologies (ATs) and Browser support it as yet. We aimed to target the most progressive supporting platform, which was Firefox 3.0 beta 5+ with Window-Eyes 6.1. Testing was also done with Firefox 2.0 and FireVox 5.3, but we elected to go with the non-namespaced role and state names, which were not supported in FireVox 5.3. The next version of FireVox will support the updated role and state names, allowing both Firefox 2.0 and Firefox 3.0 users to benefit from the ARIA support in GWT.

The details of implementing a11y support in GWT 1.5 are discussed below.

## com.google.gwt.user.client.ui.Accessibility API

This API allows DOM attributes to be added to widgets so that they can be identified by assistive technologies. It includes methods for adding, removing, and querying ARIA roles and states from a DOM element. The methods in this class are no-ops in all browser environments except FireFox 1.5+.

This class also defines constants for a subset of the ARIA role and state names.

## FocusWidget

An abstract base class for most widgets that can receive keyboard focus. A default tabIndex = 0 is applied here, to make sure that all focusable elements are in the tab sequence.

A class that inherits from FocusWidget but overrides the setElement(Element) method must make sure that tabIndex is initialized by calling on a setTabIndex(int) method.

If the inheriting class overrides setTabIndex(int) method but still inherits the setElement(Element) method from FocusWidget, the new setTabIndex(int) method will be called and nothing more needs to be done.

## CustomButton

A base button class with built in support for a set number of button faces. Inherits from ButtonBase.

CustomButton has the button role set in the constructor. The ARIA state aria-pressed - indicating when a button has been pressed down - is supported by Jaws 8.0 but not by Window-Eyes 5.5 or FireVox 5.3. Since it is not a generally supported ARIA state, it is not being used by CustomButton.

The CustomButton.onBrowserEvent(Event) method has been written to accept either the spacebar and the enter key as input to press the button. The behavior that is being modeled is Windows-specific, in that a button will not fire a click event until the space bar is being released, whereas the enter key fires on key down and also generates repeated click events if it is held down.

## MenuBar and MenuItem

A standard menu bar widget. A menu bar can contain any number of MenuItems, each of which can either fire a Command or open a cascaded menu bar.

The ARIA role menubar is applied in the MenuBar constructor, while the menuitem role is applied in the MenuItem constructor. The ARIA state aria-haspopup (which indicates whether the menu opens a cascading sub-menu) is set on those menu items with submenus.

While ARIA does have a menu role, which could be used to differentiate between a horizontal and vertical menu, this role is not supported in Window-Eyes 6.1 as yet.

GWT menu items are not naturally focusable DOM elements, so we cannot rely on focus events to indicate to the screen reader when the user navigates to a new item. To solve this sort of problem, ARIA provides the aria-activedescendant state. This state is set on the menu which has natural focus, and its value is set to the DOM id of the element that is currently selected. Whenever the user selects a different item, the state value is updated to the id of the newly-selected item. The screen reader then reads the DOM element corresponding to the id. Unique identifiers are allocated via DOM.createUniqueId().

Menus are keyboard-navigable using the arrow keys. The ENTER key is used to choose a menu item (thus causing the collapse of any pop-up menus), and the ESCAPE key is used to collapse all open pop-up menus. If the page layout is right-to-left, the arrow key navigaton will be mirrored appropriately.

When using the tab key to navigate through the controls on a page, only the root menu will appear in the tab order. To achieve this effect, tabindex = -1 is set on those menus which are submenus.

One of the problems with the keyboard implementation is that submenus do not collapse whenever the user tabs away from the menu control. Although it seems that this behaviour would be easy to implement (i.e. collapse all menus on a blur event), it is not trivial. The problem is that each menu (and sub-menu) has keyboard focus until a sub-menu is expanded. At that point, keyboard focus shifts to the sub-menu. So, blur events can occur when the user tabs away from the menu completely, OR when the user expands (or collapses) a sub-menu. If it were the case that an entire set of menus had a single focusable element (similar to the Tree) widget, or if each individual menu item was naturally focusable, then this problem could be easily solved.

## Tree and TreeItem

A standard hierarchical tree widget. The tree contains a hierarchy of TreeItems that the user can open, close, and select.

The ARIA role tree is applied in the Tree constructor, and the role treeitem is applied in the TreeItem constructor. Note that role is not applied on the root DOM element of the TreeItem; it is applied on an inner DOM element which contains the TreeItem's text.

There are a variety of ARIA states which provide information to assistive technologies about an item's position and state within a Tree. The aria-expanded state reflects whether or not the TreeItem has been expanded to show its children. The aria-level state reflects the level of the item in the tree: an item is level 1 if it has no parents, it is level 2 if it has one parent at level 1, etc. aria-posinset and aria-size reflect the position of the TreeItem in regards to its siblings and the total size of the subtree, respectively. Whenever an item is selected, the values of all of states are recomputed.

TreeItems have the same problem as MenuItems, in that they do not naturally have focus. To inform the screen reader about selection changes, the ARIA aria-activedescendant state is used. This state is set on the tree's hidden focusable element, and each TreeItem is assigned a DOM id.

Basic keyboard navigation was present, but because widgets are allowed to be added to the Tree as TreeItems, it was necessary to ensure that the tabIndex of an added widget would not interfere with the overall tab order. A widget in the Tree may come in with a set tabIndex, and this will alter the tab order of the Tree in a negative way. A user who has selected a particular TreeItem and then used tab to shift focus elsewhere in the page would find that their tab has moved their selection backward or forward to the widget, instead of outside of the Tree. When they use tab to move keyboard focus back to the Tree, their previously selected TreeItem would no longer be selected and the widget would be selected instead. Setting the tabIndex on any widgets in the Tree to be -1 avoids this issue, and the widget is still focusable by navigation through the Tree.

Keyboard navigation was also enhanced for right-to-left layout. In this layout scheme, the arrow key navigation is mirrored.

## TabBar and TabPanel

A horizontal bar of folder-style tabs, most commonly used as part of a TabPanel.

The ARIA role tablist is applied in the TabBar constructor, while the tab role is applied to each tab individually in the insertTab(String, boolean, int) method. TabPanels have the tabpanel role set in their constructors.

The ClickDelegatePanel inner class in TabBar is focusable, which allows tabs to get keyboard focus. Text or HTML-only tabs as well as tabs containing widgets are wrapped in ClickDecoratorPanels. TabBar implements both ClickListener and KeyboardListener and thus has an onClick(Widget), onKeyDown(Widget, char, int), onKeyPress(Widget, char, int), and onKeyUp(Widget, char, int) methods. Currently, onKeyDown and onKeyUp are no-ops, because onKeyPress is the most common event to handle.

Since each tab's contents are wrapped in a naturally focusable element, users can navigate between tabs using the TAB key. Generally, the method of interaction will be:

  1. press the TAB key until they select the desired tab
  1. press the ENTER key to display the tab's associated panel
  1. continue to hit the TAB key to navigate past the other tabs
  1. press TAB key once more to select a focusable control within the selected tab's associated panel

The tablist and tab ARIA roles are supported by Jaws 7.0 and later versions and Window-Eyes 5.5 and later versions. Jaws 8.0 will additionally speak "press ctrl-tab to change pages", which is incorrect because that key combination switches tabs in Firefox, not in GWT. In this case Jaws is attempting to mimic the native Windows behavior, which is not correct. Window-Eyes 5.5 will specify that a tab is a "tab control" the first time, but will not repeat the designation as the user navigates through the TabBar. If the user switches keyboard focus away from the TabBar and then back in, Window-Eyes will again specify that a tab is a "tab control".

FireVox 5.3 does not support the tablist or tab ARIA roles and will activate "caret browsing mode", which switches focus away from the TabBar and renders it unnavigable by keyboard.

A future idea would be to use the ARIA aria-selected state. It would be useful to mark the currently selected tab with selected="true".