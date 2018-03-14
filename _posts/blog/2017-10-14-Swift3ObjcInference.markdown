---
layout: post
title:  "Migrating to Swift 4"
categories: blog
excerpt: "The story of Swift 3 @objc inference"
tags: [Swift, Migration, Objective-C]
image:
feature:
date:   2017-10-14 10:39:07
modified:
share: true
---

# Migration is easy

To those of you in fear and trembling at the memory of 2->3 migration: relax. This year is so much easier. You can totally tell we're heading in the right direction of ABI compatibility in Swift 5.

After you run the migrator, double check and confirm changes it performed you're pretty much done. Oh, there's one more warning:

> The use of Swift 3 @objc inference in Swift 4 mode is deprecated. Please address deprecated @objc inference warnings, test your code with “Use of deprecated Swift 3 @objc inference” logging enabled, and then disable inference by changing the "Swift 3 @objc Inference" build setting to "Default" for the "XXX" target.

# Swift 3 @objc inference

Never heard of it? In short it lets you freely use all (convertible) Swift API from Objective-C. It converts entire API because it *assumes* you will need all of it. It is deprecated for explicitness, compiler performance and smaller executables. Read more in [the proposal][proposal].

Dropping it will make you annotate **every single** Swift declaration that is used in Objective-C code. This is huge for big codebases of mixed Swift and Objective-C.

Say you have solved all the issues and project builds - great! Some time later you're in an .m file and you want to use Swift API that has not been `@objc` annotated yet. Just go there and annotate what's needed, right? Easy when you know what you're looking for.

**When editing Objective-C files Xcode autocompletion gives no hints on non-@objc Swift API.**

API discoverablity is something you don't want to loose. Especially in cooperative projects with more than just a few extensions. Xcode won't help you here because with @objc inference turned off non-@objc API is simply not included in a generated bridging header.

So what should you do now?

## Workaround

Set `SWIFT_SWIFT3_OBJC_INFERENCE` On in build setting for debug configuration. This way Xcode autocomplition helps find a method and suggest to annotate it with a warning at the same time.

Keep Default setting for release configurations to benefit from performance optimizations. They won't build, obviously, if you disregard warnings. For best results do consider setting YES to `GCC_TREAT_WARNINGS_AS_ERRORS` flag in build setting (for all configurations).

# Extra tip

Use `@objcMembers` annotation on classes you wish their entire API available in Objective-C. No need to annotate every single method separately. This will save you tons of boilerplate noise!

(No idea why it is not mentioned in [the official migration guide][migration].)

[proposal]: https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md
[migration]: https://swift.org/migration-guide-swift4/
