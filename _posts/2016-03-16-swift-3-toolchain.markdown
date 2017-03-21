---
layout: post
title: "Adventures in Building Swift: Where's The Toolchain?"
date: 2016-03-16 10:30
comments: true
categories: tools
---
Inspired by some starter tasks being posted by Swift contributors I figured I could take a stab at what it would take to contribute. I've been writing swift for a few years now so with battle scars and a couple language migrations under my belt this past weekend I set out to get started.

# Step 1:

Follow the [Getting Started](https://github.com/apple/swift-corelibs-foundation/blob/master/Docs/GettingStarted.md) guides and get swift building locally... Cloning all the modules into the default directory layout was easy enough. Theres a script for it. SwiftFoundation includes an Xcode workspace to make development feel a little more similar to  usual Swift development. Opened it up and `⌘+R` ...

```bash
Foundation/FoundationErrors.swift:10:48: Use of undeclared type ‘ErrorProtocol'
... and many more build errors
```

# Debugging

It paid to have kept up to date on Swift's progress so I knew some renaming was going to happen with the new API [Guidelines](https://github.com/apple/swift-evolution/blob/master/proposals/0006-apply-api-guidelines-to-the-standard-library.md). Changing a few references of `ErrorProtocol` to `ErrorType` squelched some errors which clued me into what was happening. Foundation, SwiftPM, and XCTest had already migrated to the new Swift 3 style API but the latest tool chain [3/1/2016a](https://swift.org/builds/swift-2.2-branch/xcode/swift-2.2-SNAPSHOT-2016-03-01-a/swift-2.2-SNAPSHOT-2016-03-01-a-osx.pkg) at the time was still a Swift 2 Toolchain.

I struggled a bit finding some guidance that was newcomer friendly that explained how to build a whole updated Toolchain. Luckily while digging around the mailing lists I found a message referencing the following pull request. [https://github.com/apple/swift/pull/1630](https://github.com/apple/swift/pull/1630) someone else was having slightly similar issues building Swift components.

Jumping off of the mailing list traffic I built my own Toolchain

```bash
# Ran from the parent directory of the development folder because swift build system defaults depend on a particular project layout
$ swift/utils/build-toolchain local.swift
```

Then copied (or linked) the new tool chain to proper directory

```bash
# working out of the parent directory of the swift project structure
$ ln -s /swift/Library/Developer/Toolchains/swift-LOCAL-2016-03-13-a.xctoolchain /Library/Developer/Toolchains/swift-LOCAL-2016-03-13-a.xctoolchain
```

# Step 1: Part 2

Finally, again, following the normal getting started instructions I  linked Xcode and the commandline tools to the new Toolchain. Reopened the SwiftFoundation project reverting my experiment and:

```bash
Ld /Users/me/Library/Developer/Xcode/DerivedData/Foundation-bwakzktqvoihjjaxueecpzitisux/Build/Products/Debug/SwiftFoundation.framework/Versions/A/SwiftFoundation normal x86_64 ...
```
Ld is the linker and that means we just successfully build SwiftFoundation under Swift

# Done

As of 3/15/2016 there is still no prebuilt Swift 3 Toolchain so hopefully this blog post saves someone else's weekend. Some contributors have gone ahead and begin to post links to prebuilt Swift 3 Toolchains, but building your own does not take terribly long.
