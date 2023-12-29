# JavaNativeFoundation arm dylib

Provides an arm architecture dylib build of the legacy MacOS JavaNativeFoundation code. This was used by Apple and provided to developers as helper code for use in MacOS JNI development. 

## Motivation

I just switched to a arm machine when my X86 laptop had hardware issues. So I no longer have X86 available. I was going to make a minor change to an application that included JNI. This errored wanting the JNI to be arm. 

Official support by Apple is no longer available for JavaNativeFoundation. openjdk discontinued their own use of JavaNativeFoundation sometime ago so there is no support to be found there. 

## Current Availability

It is part of [apple openjdk](https://github.com/apple/openjdk/), a sort of checkpoint of Apple Java. Before they stopped supporting Java themselves. A GitHub project providing some support is [JavaNativeFoundation](https://github.com/weisJ/JavaNativeFoundation). It was unclear to me how to use this. Apple has continued to offer the related JavaVM and JavaNativeFoundation with the XCode SDK's. However, this is not official and seems to be getting more limited. 

For my own needs I tend to use simple shell script gcc invocations. Avoiding the complexities of XCode which quite a while back started making no effort to support Java in general. 

## Usage

Given the current state of support a number of workarounds seemed necessary. I had to use a older version of XCode than the version 14 I currently have. There was a strange problem at 14 that appeared to be an NSString related macro. Locating the frameworks - JavaNativeFoundation, JavaVM, also seemed more problematic with the latest. Related to the frameworks the only source changes I made to my JNI were to eliminate framework imports with includes. 

```
//#import <JavaVM/jni.h>
#include "jni.h"
```
This seemed sometimes necessary for JavaVM and sometimes (but not always?) for JavaNativeFoundation. Again, in general it is probably best to have less reliance on XCode providing these unsupported legacy frameworks. So I used include and the appropriate "-I' paths in the build.

Fortunately I had an older XCode version available from a while back when I had version related problems building the JDK. So this was not a issue for me. If you have an Apple Developer account you should be able to get one as well. 

I was going to show the different issues that made the different changes to the builds required. But I had some trouble re-creating, So, I will just show them here as-is. If you can come up with cleaner builds feel free to let me know and I will change to those.  

## HalfPipe

Again the initial motivation to consider doing this was my HalfPipe application. I had switched to an older version when it wouldn't build. But then I came across old code elsewhere I thought could be make a interesting addition to the HalfPipe scheduled functionality. The motivation to actually try to get it to build arm architecture.

I no longer provide updates to the application. It has no users I am aware of and it is to difficult for me to know without feedback if I have code that just works for me and is totally unuseable for anyone else. An application by me for me that I've had for many years.

But this is the build.

Note you need to tell it where to find the built libJavaNativeFoundation.dylib. 

```
-L/Users/mjh/Documents/JavaNativeFoundation/ -lJavaNativeFoundation \
```
For use as a jpackage application it needs to be changed to tell the application where to find it. If you include the dylib with your own in the application input directory you can do it as shown at the end of the build.

```
install_name_tool -change "libJavaNativeFoundation.dylib" "@loader_path/libJavaNativeFoundation.dylib" "libhp.dylib"
```

```
sudo xcode-select --switch /Applications/Xcode_13.1.app/
SDK_ROOT="$(xcrun --show-sdk-path)"
JDK="$(/usr/libexec/java_home)"
SDK_HDRS="$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks/Foundation.framework/Versions/C/Headers/"
SDK_FRAMEWORKS=$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks
gcc -Ijni_src -I/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/System/Library/Frameworks/JavaNativeFoundation.framework/Versions/A/Headers \
    -I${SDK_HDRS} -I${JDK}/include -I${JDK}/include/darwin \
    -L/Users/mjh/Documents/JavaNativeFoundation/ -lJavaNativeFoundation \
    -arch arm64 -o libhp.dylib -dynamiclib -F${SDK_FRAMEWORKS} -framework Foundation -framework AppKit hp_jni_src/*.m hp_jni_src/*.c
    
install_name_tool -change "libJavaNativeFoundation.dylib" "@loader_path/libJavaNativeFoundation.dylib" "libhp.dylib"`
```

## AppleScriptEngine

[AppleScriptEngine](https://github.com/mik3hall/AppleScriptEngine)

Some more unsupported legacy Apple code that I salvaged during the jdk macport project. It is the JSR 223 engine that was provided for AppleScript. The HalfPipe application has it as a dependency for limited functionality. I have verified it works there with the arm build.

```
#!/bin/bash

# In order to include JavaNativeFoundation headers this is based on 
# https://macosx-port-dev.openjdk.java.narkive.com/QZT8BXTT/is-javanativefoundation-still-supported-on-os-x-10-10
# see also
# https://github.com/apple/openjdk/
SDK_HDRS="$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks/Foundation.framework/Versions/C/Headers/"
SDK_ROOT="$(xcrun --show-sdk-path)"
JDK="$(/usr/libexec/java_home)"
SDK_HDRS="$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks/Foundation.framework/Versions/C/Headers/"
SDK_FRAMEWORKS=$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks

gcc -Inative -arch arm64 -o libAppleScriptEngine.dylib -dynamiclib \
     -I/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/System/Library/Frameworks/JavaNativeFoundation.framework/Versions/A/Headers \
     -I${SDK_HDRS} -I${JDK}/include -I${JDK}/include/darwin \
     -L/Users/mjh/Documents/JavaNativeFoundation/ -lJavaNativeFoundation \
     -F${SDK_FRAMEWORKS} -framework Foundation -framework Carbon -framework Cocoa native/*.m 
     
install_name_tool -change "libJavaNativeFoundation.dylib" "@loader_path/libJavaNativeFoundation.dylib" "libAppleScriptEngine.dylib"
```

## TRZ 

[TRZ](https://github.com/mik3hall/trz)

I wrote most of this around the introduction of Java nio.2. It overrides the default MacOS FileSystemProvider. Mostly it is fall-through to the default. The changes are to expose MacOS native file API's as nio.2 file attributes. A more recent addition was to include a JetBrains native WatchService as it's default. Replacing the JDK polling one. 

HalfPipe launches with this as it's FileSystemProvider. I plan to make this the arm jdk 21 relase leaving the jdk 20 as the x86.

```
#!/bin/bash

sudo xcode-select --switch /Applications/Xcode_13.1.app/
SDK_ROOT="$(xcrun --show-sdk-path)"
JDK="$(/usr/libexec/java_home)"
SDK_HDRS="$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks/Foundation.framework/Versions/C/Headers/"
SDK_FRAMEWORKS=$(xcode-select -p)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/System/Library/Frameworks
gcc -Ijni_src -I/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/System/Library/Frameworks/JavaNativeFoundation.framework/Versions/A/Headers \
    -I${SDK_HDRS} -I${JDK}/include -I${JDK}/include/darwin -Ifsws/include \
    -L/Users/mjh/Documents/JavaNativeFoundation/ -lJavaNativeFoundation \
    -arch arm64 -o libmacattrs.dylib -dynamiclib -F${SDK_FRAMEWORKS} -framework Foundation -framework CoreServices -framework AppKit *.m fsws/*.c
    
install_name_tool -change "libJavaNativeFoundation.dylib" "@loader_path/libJavaNativeFoundation.dylib" "libmacattrs.dylib"
```
