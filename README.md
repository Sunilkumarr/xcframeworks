# xcframeworks repo
This is a demonstration of creating and integrating the xcframeworks and their co-op with static libraries and Swift packages within the same Xcode project.

## Table of contents
* [Introduction: New .xcframework format](#Introduction:-New-.xcframework-format)
* [How to create .xcframework that contain iOS + iOS Simulator platforms](#How-to-create-.xcframework-that-contains-iOS-+-iOS-Simulator-platforms)
* [Create xcframework using fastlane plugin](#Create-xcframework-using-fastlane-plugin)
* [Testing & Troubleshooting](#Testing-&-Troubleshooting)
* [Distribution of xcframeworks](#Distribution-of-xcframeworks)
* [How to integrate .xcframework in your project](#How-to-integrate-.xcframework-in-your-project)
* [What's in XCFrameworks workspace](#What's-in-XCFrameworks-workspace)
* [Materials](#Materials)

## Pre-requisities
- Xcode 11
- Swift 5.1 toolchain - run `sudo xcode-select -s path/to/Xcode11` in terminal.
- Github/Gitlab/Bitbucket account set in Xcode's account preferences

# Introduction: New .xcframework format

## Requirements
- Xcode11
- Swift 5.1 and above

## Motivation & consequences
- introduce standard format to gain module stability for your Swift frameworks & libraries. Library author & client of a library are no longer required to use the same version of compiler
- provide seamless experience when creating & integrating the module stable frameworks
- support all Apple platforms and architectures
NOTE: while `fat framework` can support module stability, the `lipo` command line tool, that is used to fuse the frameworks together, is not officially supported by Apple. Also using `lipo` tools falls short in cases of fusing two platforms with similar architectures together - e.g. arm6 architecture can be found in iOS + watchOS.

- **STOP** creating & using `fat frameworks` == no more `lipo`.
- **STOP** slicing frameworks by stripping the architectures in your projects' targets' custom `build-phase`.

## Contents of xcframework

This format bundles module-stable frameworks (.swiftinterface) for the platforms of interest.

The [Info.plist](./Products/xcframeworks/DynamicFramework.xcframework) contains all available frameworks in a bundle specified by library identifiers. This information is used by Xcode during the linking time => xcodebuild picks the right framework for the platform we're building against

The structure of xcframework looks as shown below
![xcframework](./res/xcframework.png)

## Size of xcframework
During my tests I realized, the size of an `xcframework` was smaller than the size of an corresponding `fat framework`. I tested swift only & mixed frameworks.
Generally the `lipo` commandline tool adds a bit of overhead for all contained architectures.

## Platforms
xcframework supports all Apple platforms & their variants - `iOS`, `maccatalyst`, `macOS`, `tvOS`, `watchOS`, `iPadOS`, `carPlayOS`.

## List of destinations
| Platform  |  Destination |
|---|---|
| iOS  | generic/platform=iOS  |
| iOS Simulator  | generic/platform=iOS Simulator |
| maccatalyst | generic/platform=macOS,variant=Mac Catalyst |
| iPadOS  | generic/platform=iPadOS |
| iPadOS Simulator  | generic/platform=iPadOS Simulator|
| macOS  | generic/platform=macOS  |
| tvOS  | generic/platform=tvOS  |
| watchOS | generic/platform=watchOS |
| watchOS Simulator | generic/platform=watchOS Simulator |
| carPlayOS | generic/platform=carPlayOS
| carPlayOS Simulator | generic/platform=carPlayOS Simulator

---

# How to create .xcframework that contain iOS + iOS Simulator platforms

This section describes the process of creating the xcframework by archiving & creating the final xcframeworks from 2 archives built for `iOS` & `iOS Simulator`.

However, if you're not interested in the details of the process of `how` the xcframework is created, head directly to section: [Create xcframework using fastlane plugin](#Create-xcframework-using-fastlane-plugin).

## 1. Archive your scheme for desired platforms (destinations)
1.1 Pass `SKIP_INSTALL=NO` && `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` to archive your scheme

```swift
xcodebuild archive \
-workspace MyWorkspace.xcworkspace \
-scheme MyScheme \
-destination destination="generic/platform=iOS" \
-archivePath "archives/MyScheme-iOS" \
SKIP_INSTALL=NO \
BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

1.2 **iOS Simulator** - archive your scheme for iOS Simulator platform by specifying correct destination `destination="generic/platform=iOS Simulator"` & point archivePath to architecture specific path, e.g. `archives/MyScheme-iOS-Simulator`.

```swift
xcodebuild archive \
..
-destination destination="generic/platform=iOS Simulator" \
-archivePath "archives/MyScheme-iOS-Simulator" \
..
```

1.3 **iOS** - archive your scheme for iOS by specifying `destination="generic/platform=iOS"` & point archivePath to device specific path. The architecture specific path will ensure the archive from step 2. wont be overwritten, e.g. `MyScheme-iOS`

```
xcodebuild archive \
..
-destination destination="generic/platform=iOS" \
-archivePath "archives/MyScheme-iOS" \
..
```

### Locations
Binaries in `.xcarchive` are located under:

* `Products/Library/Frameworks` folder for dynamic frameworks
* `Products/usr/local/lib` folder for static libraries

## 2. Create .xcframework from built archives

`xcodebuild` allows you to create xcframework by specifying frameworks, libraries or even can add headers to the libraries.
![-create-xcframework](./res/xcodebuild_create_xc_framework.png)


###### 1. Specify all frameworks or libraries that you want to add into .xcframework

###### 2. Specify the outpath paht using `-output` argument. Don't forget to add `.xcframework` extension to your output path.

```
xcodebuild -create-xcframework \
           -framework My-iOS.framework \
           -framework My-iOS_Simulator.framework \
           -output My.xcframework
```

Module stability is gained with Xcode 11 + Swift 5.1, once your module declares `.swiftinterface` file, that describes the public interface of your framework along with linker flags, used toolchain and other info. Swift interface can be found under your framework's `swiftmodule` folder.
`.swiftinterface` file is autogenerated when xcframework is created.

![swift-interface](./res/swiftinterface.png)

## Create xcframework using fastlane plugin

[This plugin](https://github.com/bielikb/fastlane-plugin-create_xcframework) allows you to generate the `xcframework` (including all dSYMs & BCSymbolMaps) by specifying the desired destinations.
E.g. destination `[iOS]` will generate xcframework that contains slices for both `iOS` & `iOS Simulator`.
⚠️ Currently the plugin doesn't support static libraries.

1. Add plugin to your project
```
fastlane add_plugin create_xcframework
```

2. Add lane to your `Fastfile`
```
desc "Export xcframework"
lane :export_xcframework do
  create_xcframework(
    workspace: 'path/to/your.xcworkspace',
    scheme: 'name of your scheme',
    include_bitcode: true,
    destinations: ['iOS', 'maccatalyst'],
    xcframework_output_directory: 'Products/xcframeworks'
  )
end
```

You can try out the plugin in this project by calling following command:
```
bundle exec fastlane export_xcframework
```

---

# Testing & Troubleshooting

Make sure to always build & run your generated xcframework before distributing it to your clients.
Few of the **problems will unveil just at the compile or run time**, so **don't** rely solely on the success of the xcframework creation.

Here's the list of compiler errors I got across when integrating built xcframework into Xcode project.

| Problem  | Severity |  Description | Solution |
|---|---|---|---|
| Redundant conformance of `x` to `NSObjectProtocol` | error - thrown at dynamic linking time | Your class is already subclassed from `NSObject`, which conforms to `NSObjectProtocol`  | Check protocol conformances of your classes and remove redundant conformance to `NSObjectProtocol` |
| Use of unimplemented initializer 'init()' for class | error - thrown at dynamic linking time | Objective-C ABI public classes need to provide `public` init | Provide `public` init override for your public class:  `override public init()` |
| @objc' class method in extension of subclass of `Class X` requires iOS 13.0.0 | error | Rules for interoperability with Objective-C has changed since iOS 13.0.0. and currently doesn't support `@objc` interoperability in class extensions. There's open question on [Swift forums](https://forums.swift.org/t/xcframework-requires-to-target-ios-13-for-inter-framework-dependencies-with-objective-c-compatibility-tested-with-xcode-11-beta-7/28539) | Move/Remove `@objc` declaration from your Swift class extension |
| scoped imports are not yet supported in module interfaces | warning | Read more about Swift import declarations here: https://nshipster.com/import/ | Import the module instead of specific declaration. For example: change `import class MyModule.MyClass` to `import MyModule` |

---

# Distribution of xcframeworks
* **manually** - already available

* **Swift Package Manager**

    - [binary targets are supported since Xcode 12.0](https://developer.apple.com/wwdc20/10147)
    - define binary target in your Swift Package manifest
    - zipped xcframework filename should contain the version number
    - compute the checksum by calling `swift package compute-checksum <xcframework filename`
    - use the computed checksum in your Swift Package manifest, when referencing the xcframework remotely.


* **CocoaPods**
    - supported since v1.9.1
    - use `vendored_frameworks` to specify you xcframework(s) in your podspec. e.g. `spec.vendored_frameworks = 'DynamicFramework.xcframework'`
    - specify paths to your dSYMs and xcframework in `spec.preserve_paths = [...]`

* **Carthage**  

    - Carthage doesn't support xcframework format yet.
    - [Roadmap to provide support for xcframeworks 2019/2020](https://github.com/Carthage/Carthage/issues/2890)


---

# How to integrate .xcframework in your project

1. Drag & drop .xcframework manually into your project's target

![Drag & drop xcframework](./res/xcframework_drag_n_drop.gif)

2. Embed & sign .xcframework in your project's target

![Embed & sign .xcframework](./res/xcframework_embed_sign.png)

---

# What's in XCFrameworks workspace

`XCFrameworks` workspace consists of:
- `StaticLibrary` project - represents static library project
- `DynamicFramework` project - represents project that builds dylib
- `Swift Package` - Swift Package for internal development (within Sample project)
- `TextAttributes` - external Swift Package

- `Sample` - Sample project that includes all of the dependencies mentioned above.

![swift-interface](./res/project.png)

---

# Materials

## Binary Frameworks in Swift
https://developer.apple.com/videos/play/wwdc2019/416/

## Distribute binary frameworks as Swift packages
https://developer.apple.com/wwdc20/10147

## ABI Stability & Module Stability - swift.org
https://swift.org/blog/abi-stability-and-more/

## Library evolution in Swift - swift.org
https://swift.org/blog/library-evolution/

## Library evolution for stable ABIs
https://github.com/apple/swift-evolution/blob/master/proposals/0260-library-evolution.md

## Library evolution - Docs
https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst

## Swift Unwrapped - Swift 5.1 with Doug Gregor (Library evolution, ...)
https://spec.fm/podcasts/swift-unwrapped/308610

## Alexis Beingessner- How Swift Achieved Dynamic Linking Where Rust Couldn't
https://gankra.github.io/blah/swift-abi/

## Presentation about Dependency management in Xcode 11
https://www.slideshare.net/BorisBielik/dependency-management-in-xcode-11-153424888


## 🍕 Support the author of this repo

If you want to support me, please consider to tap on the following ad.
All my open source activities are made in my own free time, so if you like what I'm doing, feel free to support me. Thanks!

<a href="https://tracking.gitads.io/?repo=xcframeworks"><img src="https://images.gitads.io/xcframeworks" alt=“GitAds”/></a>
