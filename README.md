# gradle-android-scala-plugin

gradle-android-scala-plugin adds scala language support to official gradle android plugin.
See also sample projects at https://github.com/saturday06/gradle-android-scala-plugin/tree/master/sample

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Supported versions](#supported-versions)
- [Installation](#installation)
  - [1. Add buildscript's dependency](#1-add-buildscripts-dependency)
  - [2. Apply plugin](#2-apply-plugin)
  - [3. Add scala-library dependency](#3-add-scala-library-dependency)
  - [4. Put scala source files](#4-put-scala-source-files-optional)
  - [5. Implement a workaround for DEX 64K Methods Limit](#5-implement-a-workaround-for-dex-64k-methods-limit)
    - [5.1. Option 1: Use ProGuard](#51-option-1-use-proguard)
    - [5.2. Option 2: Use MultiDex](#52-option-2-use-multidex)
      - [5.2.1. Setup application class if you use customized one](#521-setup-application-class-if-you-use-customized-one)
- [Configuration](#configuration)
- [Complete example of build.gradle with manually configured MultiDexApplication](#complete-example-of-buildgradle-with-manually-configured-multidexapplication)
- [Changelog](#changelog)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Supported versions

| Version  | Scala   | Gradle | Android Plugin      | compileSdkVersion | comment            |
| -------- | ------- | ------ | ------------------- | ----------------- | ------------------ |
| 3.4.0    | 2.11.12 | 5.3    | 3.4.0               | 28                |                    |
| 3.3.2    | 2.11.12 | 5.3    | 3.3.2               | 28                | without submodules |
| 3.2.0-M1 | 2.11.12 | 4.9    | 3.2.0               | 28                |                    |


If you want to use older build environment,
please try [android-scala-plugin-1.3.2](https://github.com/saturday06/gradle-android-scala-plugin/tree/1.3.2)

## Installation

### 1. Add buildscript's dependency

`build.gradle`
```groovy
buildscript {
   	repositories {
   		google()
   		jcenter()
   		maven { url 'https://jitpack.io' }
   	}
   	dependencies {

   		classpath 'com.android.tools.build:gradle:3.4.0'
   		classpath 'com.github.AllBus:gradle-android-scala-plugin:3.4.0
   	}
}
```

### 2. Apply plugin

`build.gradle`
```groovy
apply plugin: "com.android.application"
apply plugin: "jp.leafytree.android-scala"
```

### 3. Add scala-library dependency

The plugin decides scala language version using scala-library's version.

`build.gradle`
```groovy
dependencies {
    implementation "org.scala-lang:scala-library:2.11.12"
}
```

### 4. Put scala source files (Optional)

Default locations are src/main/scala, src/androidTest/scala.
You can customize those directories similar to java.

`build.gradle`
```groovy
android {
    sourceSets {
        main {
            java {
                srcDirs "path/to/main/scala"
            }
        }
    }
}
```

### 5. Implement a workaround for DEX 64K Methods Limit

The Scala Application generally suffers [DEX 64K Methods Limit](https://developer.android.com/tools/building/multidex.html).
To avoid it we need to implement one of following workarounds.

#### 5.1. Option 1: Use ProGuard

If your project doesn't need to run `androidTest`, You can use `proguard` to reduce methods.

Sample proguard configuration here:

`proguard-rules.txt`
```
-dontoptimize
-dontobfuscate
-dontpreverify
-dontwarn scala.**
-ignorewarnings
# temporary workaround; see Scala issue SI-5397
-keep class scala.collection.SeqLike {
    public protected *;
}
```
From: [hello-scaloid-gradle](https://github.com/pocorall/hello-scaloid-gradle/blob/master/proguard-rules.txt)

#### 5.2. Option 2: Use MultiDex

Android comes with built in support for MultiDex. You will need to use
`MultiDexApplication` from the support library, or modify your `Application`
subclass in order to support versions of Android prior to 5.0. You may still
wish to use ProGuard for your production build.

Using MultiDex with Scala is no different than with a normal Java application.
See the [Android Documentation](https://developer.android.com/tools/building/multidex.html)
and [MultiDex author's Documentation](https://github.com/casidiablo/multidex) for
details.

It is recommended that you set your `minSdkVersion` to 21 or later for
development, as this enables an incremental multidex algorithm to be used, which
is *significantly* faster.

`build.gradle`
```groovy
repositories {
    jcenter()
}

android {
    defaultConfig {
        multiDexEnabled true
    }
}

dependencies {
    implementation "org.scala-lang:scala-library:2.11.12"
    implementation "com.android.support:multidex:1.0.3"
}
```

Change application class.

`AndroidManifest.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="jp.leafytree.sample">
    <application android:name="android.support.multidex.MultiDexApplication">
</manifest>
```

If you use customized application class, please read [next section](#521-setup-application-class-if-you-use-customized-one).

To test MultiDexApplication, custom instrumentation test runner should be used.
See also https://github.com/casidiablo/multidex/blob/publishing/instrumentation/src/com/android/test/runner/MultiDexTestRunner.java

`build.gradle`
```groovy
android {
    defaultConfig {
        testInstrumentationRunner "com.android.test.runner.MultiDexTestRunner"
    }
}

dependencies {
    implementation "org.scala-lang:scala-library:2.11.12"
    implementation "com.android.support:multidex:1.0.3"

}
```

##### 5.2.1. Setup application class if you use customized one

Since application class is executed **before** multidex configuration,
Writing custom application class has stll many pitfalls.

The application class must extend MultiDexApplication or override
`Application#attachBaseContext` like following.

`MyCustomApplication.scala`
```scala
package my.custom.application

import android.app.Application
import android.content.Context
import android.support.multidex.MultiDex

object MyCustomApplication {
  var globalVariable: Int = _
}

class MyCustomApplication extends Application {
  override protected def attachBaseContext(base: Context) = {
    super.attachBaseContext(base)
    MultiDex.install(this)
  }
}
```

**You need to remember:**

NOTE: The following cautions must be taken only on your android Application class, you don't need to apply this cautions in all classes of your app

- The static fields in your **application class** will be loaded before the `MultiDex#install`be called! So the suggestion is to avoid static fields with types that can be placed out of main classes.dex file.
- The methods of your **application class** may not have access to other classes that are loaded after your application class. As workaround for this, you can create another class (any class, in the example above, I use Runnable) and execute the method content inside it. Example:

```scala
  override def onCreate = {
    super.onCreate

    val context = this
    new Runnable {
      override def run = {
        variable = new ClassNeededToBeListed(context, new ClassNotNeededToBeListed)
        MyCustomApplication.globalVariable = 100
      }
    }.run
  }
```

This section is copyed from
[README.md for multidex project](https://github.com/casidiablo/multidex/blob/5a6e7f6f7fb43ba41465bb99cc1de1bd9c1a3a3a/README.md#cautions)

## Configuration

You can configure scala compiler options as follows:

`build.gradle`
```groovy
tasks.withType(ScalaCompile) {
    // If you want to use scala compile daemon
    scalaCompileOptions.useCompileDaemon = true
    // Suppress deprecation warnings
    scalaCompileOptions.deprecation = false
    // Additional parameters
    scalaCompileOptions.additionalParameters = ["-feature"]
}
```

Complete list is described in
http://www.gradle.org/docs/current/dsl/org.gradle.api.tasks.scala.ScalaCompileOptions.html

## Complete example of build.gradle with manually configured MultiDexApplication

`build.gradle`
```groovy
buildscript {
    repositories {
        jcenter()
        maven { url 'https://jitpack.io' }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.4.0'
        classpath 'com.github.AllBus:gradle-android-scala-plugin:3.4.0'
    }
}

repositories {
    jcenter()
}

apply plugin: "com.android.application"
apply plugin: "jp.leafytree.android-scala"

android {
    compileSdkVersion 28


    defaultConfig {
        targetSdkVersion 28
        testInstrumentationRunner "com.android.test.runner.MultiDexTestRunner"
        versionCode 1
        versionName "1.0"
        multiDexEnabled true
    }

    flavorDimensions "version"
    productFlavors {
        dev {
            minSdkVersion 21 // To reduce compilation time
            dimension "version"
        }

        prod {
            minSdkVersion 15
            dimension "version"
        }
    }

}

dependencies {
    implementation "org.scala-lang:scala-library:2.11.12"
    implementation "com.android.support:multidex:1.0.3"

}

tasks.withType(ScalaCompile) {
    scalaCompileOptions.deprecation = false
    scalaCompileOptions.additionalParameters = ["-feature"]
}
```

## Changelog
- 3.4.0 fix support submodules
- 3.3.2 Support Gradle 5.3
- 3.3.0 Support android.tools.build 3.3.0
- 3.0.0 Update zinc version

