# Building WebDriver

## Prerequisites

For all versions of WebDriver:

  * The Java JDK 8 (note that versions 9 & 10 are not currently supported for building Selenium). Download it from [Oracle's site](http://www.oracle.com/technetwork/java/javase/downloads/index.html) if it's not already on your computer.
  * Python 2.7 (note that Python 3.x is not currently supported for building Selenium).
  * The source code can be forked and/or cloned from the 
[GitHub repository](https://github.com/SeleniumHQ/selenium). Note that the repository is several gigabytes, so if you are space or bandwidth limited consider making a [shallow clone](https://git-scm.com/docs/git-clone#git-clone---depthltdepthgt).

In addition, the `InternetExplorerDriver` needs some additional components and can only be fully built on Windows

  * Visual Studio 2010 Professional or higher.
  * `"msbuild"` should be on the <var>PATH</var>. This is most easily done by starting the "Visual Studio 2010 Command Prompt" from the Start menu.

The build is based on `rake`, a well-known Ruby build tool. Provided you have all the gems required you can use that for building the project, though this isn't the recommended way to build. The bundled JRuby jar, invoked through the `go` wrapper, includes all dependencies without requiring any extra setup.

## Building

The build script will determine which parts of WebDriver will be built. When there are native components that must be built, the build file will attempt to construct them before falling back to prebuilt binaries. Assuming you have checked out the source to `$WEBDRIVER_HOME`, now:

```sh
cd $WEBDRIVER_HOME   # Where the top level Rakefile is kept
go  
# or, on a UNIX system: 
./go
```

This will not only compile the source, but will also run any tests which need to be run. If these all pass, then you have successfully built WebDriver!

There are more detailed instructions about how the build system works in the `CrazyFunBuild` section.

By default, the output of the build is somewhat terse. A log of the java compilation stages is kept in `build/build\_log.xml`. You can also get more verbose output by adding `log=true` to the build targets. eg:

`./go //java/client/test/org/openqa/selenium/support:large-tests:run log=true`

Finally, you can get even more verbose logging by modifying the Rakefile and commenting out the line that reads:

`verbose false`

The test logs are kept in the `build/test\_logs` folder.

# Tips

## Building Selenium Server

You'll get a long way with just:

`go //java/server/src/org/openqa/grid/selenium:selenium //java/client/src/org/openqa/selenium:client-combined`

That'll build the standalone "selenium server" jar and the client library too. The build output will tell you where the output files are being placed.

A shortcut for just building the standalone server is:

`go selenium-server-standalone`

The `selenium.jar` output file will be located at `buck-out/gen/java/server/src/org/openqa/grid/selenium/selenium.jar`

## Sources

`//java/server/src/org/openqa/grid/selenium:selenium-server-sources` will give you some sources.

## Building the Android Atoms
`./go //javascript/android-atoms:atoms`

# Useful Targets

All of these should be run using the "go" executable.

For everything:

| **Target** | **Purpose** |
|:-----------|:------------|
| clean      | Delete all built artefacts |

For Java:

| **Target** | **Purpose** |
|:-----------|:------------|
| test\_java | Run every applicable Java test |
| test\_htmlunit | Run all the HtmlUnitDriver tests |
| test\_ie   | Run all the InternetExplorerDriver tests |
| test\_firefox | Run all the FirefoxDriver tests |
| selenium-server-standalone | Build the standalone server |
| //java/client/test/org/openqa/selenium:single:run | Run the SingleTestSuite |

Useful parameters:

| **Parameter** | **Purpose** | **Example**|
|:--------------|:------------|:-----------|
| haltonerror   | This flag indicates if "go" should halt at an error. Default is true. | ./go test\_firefox haltonerror=false |
| haltonfailure | This flag indicates if "go" should halt at a failure. Default is true. | ./go test\_firefox haltonfailure=false |
| method        | If set, only the specified test will be executed. | ./go test\_firefox method=testDoubleClick |
| onlyrun       | If set, only tests in the specified class will be executed. | ./go test\_firefox onlyrun=XPathElementFindingTest |
| log           | This flag indicates logging (e.g. test result streaming) should be dumped to the console, not just files. Default is false. | go test\_firefox log=true |
| suspend       | If set, suspends the JVM until a debugger connects before running tests.  Useful in conjunction with debug. Default is false. | ./go test\_firefox suspend=true debug=true |
| debug         | Enable the java remote debugger when starting JVMs for tests.  Useful in conjunction with suspend. Default is false. Default debugger port is 5005. | ./go test\_firefox suspend=true debug=true |
| offline       | If set, don't try to download any components from the internet (e.g. the gecko sdk). Default is false. | ./go test\_firefox offline=true |
| leaverunning  | If set the browser window will not be closed after the last test. | ./go test\_firefox leaverunning=true |
| headless      | Only applies to the android emulator.  If set, the android emulator window will not be opened (and so the test run will not require an X display) | ./go test\_android headless=true |

For Javascript:

| **Target** | **Purpose** |
|:-----------|:------------|
| //jsapi:debug:run | Run the test server (useful for working) |
| calcdeps   | Recalculate dependencies once a `goog.provide` or `goog.require` changes |

# Common Problems

## Nothing Compiles

Make sure that you've got the <var>JAVA\_HOME</var> environment setting properly set up. In addition, make sure that you can execute `rake`, `java`, `jar`, and `javac` from the command line. If you're on Windows, you'll also need to be able to execute `devenv`. You may need to install a JDK. If a C++ component doesn't build properly on Windows, make sure that you are actually using the VS2010 version of msbuild (this might not be the case, even if you're in a VS2010 Command Prompt)

## All Firefox Tests Fail

Occasionally we change the protocol of how the java code in the Firefox driver talks to the extension. The main reason why all the firefox tests fail is because the user has installed a profile called "WebDriver". This should be deleted.

The following trouble-shooting steps might be useful:

  1. Check that you have the latest version of the code (`svn up`)
  1. Delete the WebDriver profile. The easiest way to do this is to start firefox using `firefox -ProfileManager`

Although it was necessary in the early days of webdriver development, it is now no longer necessary to install the webdriver extension manually. Indeed, doing so is more likely to be the root of hard to track down errors. It is strongly recommended that you **do not** install the Firefox extension manually

## The Build Works, but It's Very Slow

There have been reports of problems involving slow builds. Every test that runs does a DNS lookup to determine an alternative host name. If your network is not configured correctly, then this lookup might be very slow. To rectify this, modify the `getAlternateHostName` method in `org.openqa.selenium.environment.webserver.JettyAppServer` and hard code it to return a string that resolves to your machine that isn't `localhost`.

## I've Followed the Steps Above and All the Firefox Tests Still Fail

Other things to check:

  * The `FirefoxDriver` is only compatible with Firefox 3 and above. Check that the first available version of Firefox on the <var>PATH</var> is version 3 or later.
  * The driver assumes that Firefox is installed in the default location for your OS. If Firefox is not in this location, then you need to set the VM property `webdriver.firefox.bin` or modify the <var>PATH</var> variable to include the directory with the Firefox binary in it.
  * On some platforms such as Linux, Firefox is started with a shell script. There have been reports that if your installation of Firefox wraps this shell script with another one the `FirefoxDriver` won't work properly. Consider calling the original Firefox script.

# When do the prebuilds need re-building?

  * Most changes to the atoms will lead to a new `cpp/IEDriver/Generated/atoms.h` file.  If this is the case, the `IEDriver` prebuilds needs recompiling, in a Visual Studio command prompt on Windows.
  * The iPhone driver, and ChromeDriver will also be affected by these changes, but let's be honest here, you probably aren't going to do anything about those.
  * Any change to the C++ under cpp will either need the `IEDriver`, `firefox-driver`, or both to be rebuilt.

## Building the Firefox prebuilts on Linux

  * Get yourself a 64 bit machine running Ubuntu
  1. Install the following: .... (list TBD)
  1. Run `./go firefox`
  1. Update the index to tell git that the files are modified (so you can `git add` them) `git update-index --no-assume-unchanged cpp/prebuilt/*/libwebdriver_firefox_*`

  * use `lxc` to create an instance of Ubuntu for 32-bit (i386)
  1. And repeat steps above ^