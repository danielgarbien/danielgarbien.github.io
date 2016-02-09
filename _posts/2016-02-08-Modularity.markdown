---
layout: post
title:  "Modularity in iOS apps"
date:   2016-02-08 19:39:07
categories: ios
---

Too many times I was loosing sleep over unraveling biderectional dependencies between view controllers in an aged codebase. Greatly inspired by [On Monoliths and Microservices][otto] by Guido Steinacker paired with my own work experience I've decided to underline a modularity issue in iOS app. There is no such project kind that deos not grow and evolve. An urge to code pragmatic shortcuts turns out to be as much appealing as lethal at a (sooner than) later stage.

Even though the post relates to a server side development, the idea is universal. "Share nothing architecture" implemented to some extent makes sense in a front dev to.
Be smart from day one. Split your app in modular, testable parts.

## The architecture

We could create a separate subproject for each screen, but this might sometimes be an overkill. Instead, let's split a project in a more virtual manner. We aim for an architecture model setup premising the following:

* Project's implementation is split into modules (forgive a name collision with [modules][apple]) where each one has no idea whatsoever about another
* The flow and connections between them is controlled by superordinate objects
* Each module can be easily run and debugged on its own
* Each module can be tested with UI tests

Now let's work them one by one.

## Split into modules

We can achieve it by placing module files in separate groups/folders. This kind of "horizontal" grouping stands in opposition to more frequently met "vertical" grouping into Models/Views/Controllers. Mind the virtuality of this solution. It is only up to you to keep the separation clean.

![Alt text][groups]

## Controlling the flow

Wireframe instance is the guy in charge (see [Viper architecture][viper]). It is not a UIViewController itself, but it manages interactions between suburdinate controllers. It is usually their delegate. On top of that a main wireframe handles loading a root view controller in a window.

## Launching modules on their own

This is a tricky part. We're going to hire an additional application delegate -LaunchModuleAppDelegate. It'll only be used when special condition is met. Let's add main.swift with:

``` swift
if (LaunchModule.currentProcessLaunchModule() != nil){
    UIApplicationMain(Process.argc, Process.unsafeArgv, NSStringFromClass(UIApplication), NSStringFromClass(LaunchModuleAppDelegate))
} else {
    UIApplicationMain(Process.argc, Process.unsafeArgv, NSStringFromClass(UIApplication), NSStringFromClass(AppDelegate))
}
```

This special condition is a "launchModule" environmental variable added to the scheme. Below you see how easy it is to switch between running an entire project/single module alone and vice versa!

<img src="https://cloud.githubusercontent.com/assets/3668771/12930465/c1c550e4-cf78-11e5-9ee2-e7dc8233b410.png" alt="Environment variable" style="width: 600px;"/>

Anticipated values are defined in LaunchModule.swift:

``` swift
/// Launch module environment variable name
private let launchModuleEnvironmentVariableName = "launchModule"
/// Launch module expected environment value names
enum LaunchModule: String {
    case Master = "master"
    case Detail = "detail"
}
``` 

The code for retrieving the actual launchModule value lives in the same file:

``` swift
extension LaunchModule {
    static func currentProcessLaunchModule() -> LaunchModule? {
        return NSProcessInfo.launchModule()
    }
}
private extension NSProcessInfo {
    static func launchModule() -> LaunchModule? {
        guard let launchModule = processInfo().environment[launchModuleEnvironmentVariableName] else {
            return nil
        }
        return LaunchModule(rawValue: launchModule)
    }
}
``` 

LaunchModuleAppDelegate skips the Wireframe and loads a specific module of choice as a root view controller. Thanks to using a separate app delegate the implementation of the original AppDelegate remains untouched. Bless the separation of concerns.

## UI tests

In order to make a XCTestCase subclass be module restricted it needs to make an app launch with a specific module. Again, this is possible with use of environment variables passed to XCUIApplication at a setUp stage:

``` swift
class DetailUITests: XCTestCase {
    override func setUp() {
        super.setUp()
        XCUIApplication(launchModule: .Detail).launch()
    }
    func testDetailScreen() {...}
}
```

Convenience initializer is added in XCUIApplication extension:

``` swift
extension XCUIApplication {
    convenience init(launchModule: LaunchModule) {
        self.init()
        launchEnvironment = launchModule.launchEnvironment()
    }
}
```

Generation of launchEnvironment dictionary is implemented in LaunchModule so the key/value logic is kept in one place. LaunchModule therefor must be added to both project's and UI tests targets.

``` swift
extension LaunchModule {
    func launchEnvironment() -> [String: String] {
        return [launchModuleEnvironmentVariableName: rawValue]
    }
}
```
&nbsp;
## Voilà

There, we did it! Each module is not aware of others, nor about the flow. It's runnable and compatible with automated tests.

Demo project on [github][github].

[otto]: http://dev.otto.de/2015/09/30/on-monoliths-and-microservices/
[apple]: https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AccessControl.html
[viper]: https://www.objc.io/issues/13-architecture/viper/
[groups]: https://cloud.githubusercontent.com/assets/3668771/12929088/45cd2446-cf71-11e5-8a5b-1464d531a5cc.png
[github]: https://github.com/danielgarbien/Modularity
