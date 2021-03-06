**Warning: This information is related to legacy driver implementations (2.0), for W3C WebDriver compatible implementations see https://w3c.github.io/webdriver/webdriver-spec.html#actions**

# Introduction

The Advanced User Interactions API is a new, more comprehensive API for describing actions a user can perform on a web page. This includes actions such as drag and drop or clicking multiple elements while holding down the Control key.

## Getting started (short how-to)
In order to generate a sequence of actions, use the [Actions](https://github.com/SeleniumHQ/selenium/blob/master/java/client/src/org/openqa/selenium/interactions/Actions.java) generator to build it. First, configure it:

```java
   Actions builder = new Actions(driver);

   builder.keyDown(Keys.CONTROL)
       .click(someElement)
       .click(someOtherElement)
       .keyUp(Keys.CONTROL);

```

Then get the action:

```java
   Action selectMultiple = builder.build();
```

And execute it:

```java
   selectMultiple.perform();
```

The sequence of actions should be short - it's better to perform a short sequence of actions and verify that the page is in the right state before the rest of the sequence takes place. The next section lists all available actions and how can they be extended.

## Keyboard interactions
Until now, keyboard interaction took place through a specific element and WebDriver made sure the element is in the proper state for this interaction. This mainly consisted of scrolling the element into the viewport and focusing on the element.

Since the new Interactions API takes a user-oriented approach, it is more logical to explicitly interact with the element before sending text to it, like a user would. This means clicking on an element or sending a `Keys.TAB` when focused on an adjacent element.

The new interactions API will (first) support keyboard actions without a provided element. The additional work to focus on an element before sending it keyboard events will be added later on.

## Mouse interactions
Mouse actions have a context - the current location of the mouse. So when setting a context for several mouse actions (using `onElement`), the first action will be relative to the location of the element used as context, the next action will be relative to the location of the mouse at the end of the last action, etc.

# Current status
The API is (mostly) finalized for the actions and actions generator. It is fully implemented for HtmlUnit and Firefox and in the process of being implemented for [Opera](OperaDriver.md) and IE.

# Outline
## A single action
All actions implement the [Action](https://github.com/SeleniumHQ/selenium/blob/master/common/src/java/org/openqa/selenium/interactions/Action.java) interface. This action only has one method: `perform()`. The idea being that each action gets the required information passed in the Constructor. When invoked, the action then figures out how it should interact with the page (for example, finding out the active element to send the key to or calculating the screen coordinates of an element for a click) and calls the underlying implementation to actually carry out the interaction.

There are currently several actions:

  * `ButtonReleaseAction` - Releasing a held mouse button.
  * `ClickAction` - Equivalent to `WebElement.click()`
  * `ClickAndHoldAction` - Holding down the left mouse button.
  * `ContextClickAction` - Clicking the mouse button that (usually) brings up the contextual menu.
  * `DoubleClickAction` - double-clicking an element.
  * `KeyDownAction` - Holding down a modifier key.
  * `KeyUpAction` - Releasing a modifier key.
  * `MoveMouseAction` - Moving the mouse from its current location to another element.
  * `MoveToOffsetAction` - Moving the mouse to an offset from an element (The offset could be negative and the element could be the same element that the mouse has just moved to).
  * `SendKeysAction` - Equivalent to `WebElement.sendKey(...)`

The `CompositeAction` contains other actions and when its perform method is invoked, it will invoke the perform method of each of the actions it contains. Usually, the actions should not created directly - the `ActionChainsGenerator` should take care of that.

## Generating Action chains
The `Actions` chain generator implements the [Builder](http://en.wikipedia.org/wiki/Builder_pattern) pattern to create a `CompositeAction` containing a group of other actions. This should ease building actions by configuring an `Actions` chains generator instance and invoking its `build()` method to get the complex action:

```java
   Actions builder = new Actions(driver);

   Action dragAndDrop = builder.clickAndHold(someElement)
       .moveToElement(otherElement)
       .release(otherElement)
       .build();

   dragAndDrop.perform();
```

A planned extension to the `Actions` class is adding a method that will append any Action to the current list of actions it holds. This will allow adding extended actions without manually creating the `CompositeAction`. On extending actions, see below.

## Guidelines for extending the Action interface
Thie `Action` interface only has one action - `perform()`. In addition to the actual interaction itself, any evaluation of conditions should be performed in this method. It's possible that the page state has changed between creation of the action and when it was actually performed - so things like element's visibility and coordinates shouldn't be found out in the `Action` constructor.

# Implementation details
To achieve separation between the operations each action is performing and the actual implementation of the operations, all actions rely on two interfaces: `Mouse` and `Keyboard`. These interfaces are implemented by every driver that supports the advanced user interactions. Note that these interfaces are designated to be used by the actions - not by end users - the information in this section is only useful for developers planning to extend WebDriver.

## A word of warning
The keyboard and mouse interface are designed to be used by the various action classes. For this reason, their API is less stable than that of the `Actions` chain generator. Directly using these interfaces may not yield the expected results, as the actions themselves do additional work to make sure the right conditions are met before events are actually generated. Such preliminary work includes focusing on the right element or making sure the element is visible before any mouse interaction.

## Keyboard
The `Keyboard` interface has three methods:
  * `void sendKeys(CharSequence... keysToSend)` - Similar to the existing `sendKeys(...)` method.
  * `void pressKey(Keys keyToPress)` - Sends a key press only, without releasing it. Should only be implemented for modifier keys (<kbd>Control</kbd>, <kbd>Alt</kbd>, and <kbd>Shift</kbd>).
  * `void releaseKey(Keys keyToRelease)` - Releases a modifier key.

It is the implementation's responsibility to store the state of modifier keys between calls. The element which will receive those events is the active element.

## Mouse
The `Mouse` interface includes the following methods (This interface will change soon):
  * `void click(WebElement onElement)` - Similar to the existing `click()` method.
  * `void doubleClick(WebElement onElement)` - Double-clicks an element.
  * `void mouseDown(WebElement onElement)` - Holds down the left mouse button on an element.
> > Action selectMultiple = builder.build();
  * `void mouseUp(WebElement onElement)` - Releases the mouse button on an element.
  * `void mouseMove(WebElement toElement)` - Move (from the current location) to another element.
  * `void mouseMove(WebElement toElement, long xOffset, long yOffset)` - Move (from the current location) to new coordinates: (X coordinates of <var>toElement</var> + <var>xOffset</var>, Y coordinates of <var>toElement</var> + <var>yOffset</var>).
  * `void contextClick(WebElement onElement)` - Performs a context-click (right click) on an element.

## Native events versus synthetic events

In WebDriver advanced user interactions are provided by either simulating the Javascript events directly (i.e. synthetic events) or by letting the browser generate the Javascript events (i.e. native events). Native events simulate the user interactions better whereas synthetic events are platform independent, which can be important in Linux when alternative window managers are used, see [native events on Linux](NativeEventsOnLinux.md). Native events should be used whenever it is possible.

The following table shows which browsers support which kind of events:

| **Browser** | **Operating system** | **Native events** | **Synthetic events** |
|:------------|:---------------------|:------------------|:---------------------|
| Firefox     | Linux                | supported         | supported (default)  |
| Firefox     | Windows              | supported (default) | supported            |
| Internet Explorer | Windows              | supported (default) | not supported        |
| Chrome      | Linux/Windows        | supported`*`      | not supported        |
| Opera       | Linux/Windows        | supported (default) | not supported        |
| HtmlUnit    | Linux/Windows        | supported (default) | not supported        |

`*`) `ChromeDriver` provides two modes of supporting native events, called WebKit events and raw events. In the WebKit events the `ChromeDriver` calls the WebKit functions which trigger Javascript events, in the raw events mode operating systems events are used.

In the `FirefoxDriver`, native events can be turned on and off in the `FirefoxProfile`.

```java
FirefoxProfile profile = new FirefoxProfile();
profile.setEnableNativeEvents(true);
FirefoxDriver driver = new FirefoxDriver(profile);
```

### Examples

These are some examples where native events behave different to synthetic events:

  * With synthetic events it is possible to click on elements which are hidden behind other elements. With native events the browser sends the click event to the top most element at the given location, as it would happen when the user clicks on the specific location.
  * When a user presses the <kbd>tab</kbd> key the focus jumps from the current element to the next element. This is done by the browser. With synthetic events the browser does not know that the <kbd>tab</kbd> key is pressed and therefore won't change the focus. With native events the browser will behave as expected.
