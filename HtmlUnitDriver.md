# HtmlUnit Driver

This is currently the fastest and most lightweight implementation of WebDriver. As the name suggests, this is based on HtmlUnit.

## Pros

  * Fastest implementation of WebDriver
  * A pure Java solution and so it is platform independent.
  * Supports Javascript

## Cons

  * Emulates other browser's JS behaviour (see below)

## Javascript in the HtmlUnitDriver

None of the popular browsers uses the Javascript engine used by `HtmlUnit` ([Rhino](http://www.mozilla.org/rhino/)). If you test Javascript using `HtmlUnit` the results may differ significantly from those browsers.

When we say “javascript”, we actually mean “javascript and the DOM”. Although the DOM is [defined by the W3C](http://www.w3.org/DOM/), each browser out there has its own quirks and differences in their implementation of the DOM and in how javascript interacts with it. 

HtmlUnit has an impressively complete implementation of the DOM, and has good support for using Javascript. But it is no different from any other browser; it has its own quirks and differences from both the W3C standard and the DOM implementations of the major browsers, despite its ability to [mimic other browsers](http://htmlunit.sourceforge.net/javascript.html).

With WebDriver, we had to make a choice; do we enable HtmlUnit's Javascript capabilities and run the risk of teams running into problems that only manifest themselves there, or do we leave Javascript disabled, knowing that there are more and more sites that rely on Javascript? We took the conservative approach, and by default have disabled support when we use HtmlUnit. With each release of both WebDriver and HtmlUnit, we reassess this decision: we hope to enable Javascript by default on the HtmlUnit at some point.

## Enabling Javascript

If you can't wait, enabling Javascript support is very easy:

```java
HtmlUnitDriver driver = new HtmlUnitDriver();
driver.setJavascriptEnabled(true);
```

or

```java
HtmlUnitDriver driver = new HtmlUnitDriver(true);
```

This will cause the HtmlUnitDriver to emulate IE's Javascript handling by default.

## Emulating a Specific Browser

Notwithstanding other considerations above, it is possible to get HtmlUnitDriver to emulate a specific browser.  You should not really be doing this, as web-applications are better coded to be neutral of which reasonably recent browser you are using.  There are two more constructors for `HtmlUnitDriver` that take allow us to indicate a browser to emulate. One takes a browser version directly:

```java
HtmlUnitDriver driver = new HtmlUnitDriver(BrowserVersion.FIREFOX_3);
```

The other uses a broader capabilities mechanism:

```java
HtmlUnitDriver driver = new HtmlUnitDriver(capabilities);
```