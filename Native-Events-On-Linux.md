# Introduction

This page details how native events for Firefox on Linux were implemented. This feature allows better user emulation for WebDriver in Firefox on Linux. Using this implementation, it should be easily extended to any other browser with a plug-in mechanism that is implemented using GTK+.

# Window Managers known to break with Native Events

Not all window managers were born equal. Specifically, because of focus stealing, some scenarios of window switching will fail with some window managers.

## Window managers known to work

  * GNOME
  * KDE

## Window managers known not to work

  * Enlightment / E16 - Switching to a newly opened window is broken.
  * TWM - The same as Enlightment / E16.
  * AfterStep - The same as Enlightment / E16.
  * WindowMaker - Breaks on focus remaining in the original window when opening a new one. The reason: In Gnome, when a sub-window of the new window gets focus this triggers focus stealing. This does not happen in WindowMaker, due to it not having a "smart" focus policy, where new windows get focus.

# Purpose and scope

## Existing State

WebDriver offers user emulation via Javascript on all platforms, but on Windows "native events" are used. Specifically, sending keys to Firefox on Windows involves generating the Windows-specific events to make Firefox think it received user input. The same should be possible on Linux. As on Windows, it should be possible to run more than one instance of the Firefox driver at a time without the two interfering with each other. The final requirement, which is what makes native events on every platform challenging, is that the browser should not require focus in order to work properly --- it's useful to read email while the tests are running.

## Technical knowledge required

In order to understand the underlying mechanism of calling native code from within a Firefox extension, familiarity with XPCOM is recommended. The following links were useful:
  * https://developer.mozilla.org/en/Creating_XPCOM_Components/An_Overview_of_XPCOM
  * http://blog.mozilla.com/addons/2009/01/28/how-to-develop-a-firefox-extension/


# Possible solutions surveyed

## X11 Events

The basic approach is creating a raw _XEvent_ and send it to the appropriate window using _XSendEvent_. Unfortunately, this approach will not work because synthetic events are marked, so applications receiving them actually know that those are generated events and usually ignore them. A suggestion on how to over-ride this problem was suggested by [semicomplete.com](http://www.semicomplete.com/blog/geekery/xsendevent-xdotool-and-ld_preload.html) : Over-ride XNextEvent (which is the function used to pop the next X event from the send queue) to pop the next event and turn off the _send\_event_ flag. While this solves the problem of synthetic events, it does not solve another problem: Some applications, including Firefox, will simply ignore events when in the background.

## XTest

Xtest is the most straightforward way to send X input events. It is used by many project to emulate user input or to translate input from a non-standard device to X windows input. For example, [Synergy](http://synergy2.sourceforge.net/) uses it. The problem with this solution is, again, that the window requires the focus to get those evnts.

### XTest + Xvnc

A possible way to overcome this disadvantage would be to run Firefox inside an Xvnc server in which it'll have all the focus it wants. This means packing more stuff with WebDriver or forcing our user to have some more pre-requisites for running WebDriver. For these reasons, it was frowned upon by SimonStewart, so we didn't go this way. **Update:** It looks like running Firefox 3 in Xvnc is problematic (Firefox will exit with some X error).

## GDK Events / GTK Signals

Under Windows, the native code receives a window handle to the current window from the accessibility API (See [NsIAccessibleDocument](https://developer.mozilla.org/en/NsIAccessibleDocument)). On Linux, the same API yields a GdkWindow**. There are two ways from there:
  * The GTK way.
  * The GDK way.**

A distinction between GTK+ and GDK is required: GDK is a slim wrapping over the raw, painful X API. GTK+ (which uses GDK) is a complete toolkit by itself, supplying a whole set of widgets for GUI application builders to use.

The GTK way would be to create GTK signals. The GDK way would be creating GdkEvents and pushing them into the queue that (normally) gets the translated X events. Since we're getting a GdkWindow**, it only makes sense we'll go the GDK way, doesn't it?**

# Implementation

## Over-coming the focus problem

All of the methods above had the same problem: Firefox, when not having the focus, ignores events sent to it. Just as when trying to feed a four years-old child, it is inappropriate to try and shove food down his throat when it is not paying attention, so trying to force Firefox to handle events when in the background is deeply futile.

Like with a four years-old, trickery will get you much further: trick Firefox into constantly thinking it's in the foreground by never letting the `FocusOut` events reach it. X events are captured in the same way described above (See [Events](NativeEventsOnLinux#X11.md)) and when a FocusOut detected it is replaced by another event. The exact code can be found in https://github.com/SeleniumHQ/selenium/blob/master/cpp/linux-specific/x_ignore_nofocus.c

## GDK Events

GDK has two relevant events: `GDK\_KEY\_PRESS` and `GDK\_KEY\_RELEASE` which are created using `gdk\_event\_new`. After filling out the relevant event information (mainly the key pressed and a timestamp), it's pushed into the GDK event queue using `gdk\_event\_put`.
Notes worth paying attention to:
  * The window handle received is `GdkWindow`**. It must be assigned into the `window` field of the event** after being referenced (`_g\_object\_ref`), as it will be de-referenced when the event is deleted.
  * Events are copied into the queue. They must be released afterwards using `_gdk\_event\_free`.
  * **Events will not be processed until after the native call has returned to the Javascript code and some time has passed, For this reason, the Javascript code must sleep for a bit (which is why it'll only work in Firefox 3).**

## Event Generation_

### Filling the event fields

The following fields are being filled:
  * _window_ <- the window handle.
  * _send\_event_ <- 0, as we baldly lie about it not being a synthesized event.
  * _time_ <- current time since system boot in milliseconds.
  * _hardware\_keycode_ <- The actual hardware key code for the letter it represents. This may be different according to different keyboard layouts and is only filled for the sake of completeness. See _XKeysimToKeycode_.
  * _keyval_ <- the value of the character to be emulated. If it's a regular character it's translated via gdk\_unicode\_to\_keyval. If it's a modifier or non-alphanumeric key it gets the GDK symbol (like GDK\_Shift\_L, for example).
  * _is\_modifier_ <- gets `1` if it's a modifier key, `0` otherwise. Makes sense?
  * _state_ <- OR-ed mask representing all of the "pressed" modifier keys.

### Event generation algorithm

  * Generally, characters are sent as-is. This means two events are being generated for each character: Key Press, Key Release.
  * Modifier keys may also be sent (Such as _sendKeys(Keys.CTRL, "a")_). In this case, only one key event will be generated - either Key Press or Key Release, depending on the previous state of the modifier.
  * If the character can only be generated by using the shift key (such as uppercase characters or some punctuation marks), A shift key press is emulated (Shift key press before the character and key release afterwards).
  * If the Shift modifier was set when we receive an uppercase character key, that will not apply. The end result will still be an uppercase letter.
  * If the Null key was sent, it clears the modifiers and generates no events.
  * At the end of a `sendKeys` call, all of the modifiers that were pressed will be released - the appropriate release events will be generated.

# Detailed implementation details

## Files

The following files are relevant to the implementation:
  * **common/src/cpp/webdriver-interactions/interactions\_linux.cpp** - All of the native events creation code.
  * **firefox/src/cpp/linux-specific/x\_ignore\_nofocus.c** - The code that hacks XNextEvent never to return focus out events. This code is used to generate a shared object that "injects" a hacked version of XNextEvent into Firefox (using LD\_PRELOAD).
  * **firefox/src/cpp/webdriver-firefox/native\_events.cpp** - Entry point for the native method calls. While this has no platform-specific code in theory, it has some, in practice.

## Translation between Keys.java and GDK key symbols

This is done using a function called _translate\_code\_to\_gdk\_symbol_ which is auto-generated using some python code. At the moment, it's not in the repository. What it does is produce the appropriate GDK key symbol for a given key code from Keys.java.

## XModifierKey

A class representing a modifier key and holds the GDK key symbol for this key, the mask representing this modifier (the thing that should be OR-ed to the _state_ field of the GdkEvent) and the actual state of the modifier.

## KeypressEventsHandler

The class that actually is responsible for generating the Gdk Events. The most important method is:

```java
list<GdkEvent*> CreateEventsForKey(wchar_t key_to_emulate);
```

Which returns a list of event (which, in turn, will be submitted and freed) per key to emulate.

The other two public methods:

```java
list<GdkEvent*> CreateModifierReleaseEvents();
```

Which is called before the sendKeys method returns - to generate events that indicate releasing all of the modifier keys that were set.

And:

```java
guint32 get_last_event_time();
```

Which returns the time-stamp associated with the last event that was created. This is needed for another native call, _pending\_keyboard\_events()_ which will be called from the Javascript code **after** `sendKeys` has returned, to make sure all of the events were handled and removed from the queue. That is the only way to make sure that the Javascript code that does the `sendKeys` call will not return too soon (before all key press / release events have arrived to the window). See `utils.js` for more details.

# Some troubleshooting tips
The first step when suspecting native events do not work is to verify the libraries needed are loaded (both for focus stealing and for the library itself). The following command should be executed when there's a Firefox instance that was started by WebDriver:

```sh
[user@host:~,1]$ for i in `pidof firefox`; do cat /proc/${i}//maps |grep "libwebd\|ignore"; done
7f53791f3000-7f537920d000 r-xp 00000000 08:01 2551370                    /tmp/webdriver8142511255007268151profile/extensions/fxdriver@googlecode.com/platform/Linux_x86_64-gcc3/components/libwebdriver-firefox.so
7f537920d000-7f537940c000 ---p 0001a000 08:01 2551370                    /tmp/webdriver8142511255007268151profile/extensions/fxdriver@googlecode.com/platform/Linux_x86_64-gcc3/components/libwebdriver-firefox.so
7f537940c000-7f537940e000 rw-p 00019000 08:01 2551370                    /tmp/webdriver8142511255007268151profile/extensions/fxdriver@googlecode.com/platform/Linux_x86_64-gcc3/components/libwebdriver-firefox.so
7f538c00b000-7f538c012000 r-xp 00000000 08:01 2551405                    /tmp/webdriver8142511255007268151profile/amd64/x_ignore_nofocus.so
7f538c012000-7f538c212000 ---p 00007000 08:01 2551405                    /tmp/webdriver8142511255007268151profile/amd64/x_ignore_nofocus.so
7f538c212000-7f538c213000 rw-p 00007000 08:01 2551405                    /tmp/webdriver8142511255007268151profile/amd64/x_ignore_nofocus.so
```

Note that **both** `x\_ignore\_nofocus` and `libwebdriver-firefox` should appear on the output. If one (or both) are missing, then native events will not function correctly (or at all).
