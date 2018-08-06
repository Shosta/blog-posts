---
title: Set up your Android app for Offensive Security
published: false
description: How to setup an Android app for assessing its security and start to perform some security testing on it.
tags: Security, Android, PenetrationTesting, Go
---

As I mentioned in one of my previous article about Android Security, the first part of any security assessment is to gather information about the application.
{% link https://dev.to/shosta/offensive-security-on-an-android-app-5e06 %}

----

It is very easy to do it on a single application when you know the proper tools and how to use them.

But if you do it on various applications it quickly become hard work.
If you are a software engineer (and I’m sure that you are), as soon as you are doing something more than 5 times, it is good to automate it. 🖥

That’s why I automated all that setup before I can work on an app.

----

> *What’s that setup?*

It is very simple. Let’s consider that you don’t have a rooted android terminal (and you don’t need one for security testing at this point), you must just perform some changes to the application.

Making it available for debugging, allowing backup, removing Certificate pinning.
Once it’s done, you are good to go.

----

> *How do I do that?*

Let’s pull the app you want to work on from your android phone.

You want to know the package name of the app you want to work on, so list all the packages and look for the one you want to work on.

```bash
> adb shell pm list packages
```
<center><h6>List all the packages on your android device</h6></center>


But it can lead to a massive amount of packages to look into.
Let’s use the magical grep command.


```bash
> adb shell pm list packages | grep insta
```
<center><h6>Let's say that you are looking for the instagram packages and assume that it has "insta" in it</h6></center>

Now we need the path to the package on our android device.

```bash
> adb shell pm path com.instagram.android
```
<center><h6>Let use another popular adb command to retieve the pacakge path on the android device</h6></center>

And pull from the android device to the computer using another adb command.

```bash
> adb shell pull /pkg/path/on/device/pkg.apk /dest/on/computer
```
<center><h6>Pull the apk from the android device to the computer</h6></center>

----

> *Good, now I have the apk on my computer. But what to do now?*

Disassemble the app may be a good idea. 😉
I’ll assume that you installed apktool and put it in your PATH.

```bash
> apktool d /path/to/pkg.apk -f -o /path/to/disassemble/apk/
```
<center><h6>Disassemble the package using apktool</h6></center>

You should have something like this in your folder.

![A classic disassembled package folder](https://cdn-images-1.medium.com/max/800/1*A2d4BApKHT0cI6KM_Coi2A.png)
<center><h6>A classic disasembled package folder</h6></center>


> *Ok, I disassembled the package. How could I modified the application?*

Everything that we are going to modify is located in the AndroidManifest.xml file upon the application.

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:installLocation="auto" android:targetSandboxVersion="2" package="com.package">
    <uses-permission android:name="android.permission.CAMERA"/>
    ...
    <application android:icon="@mipmap/ic_launcher" android:label="@string/main_app_name" android:largeHeap="true" android:name="com.android.packagename" android:networkSecurityConfig="@xml/network_security_config" android:supportsRtl="false" android:theme="@style/Theme.App">
        <meta-data android:name="tapad.APP_ID" android:value="package-id-tracker"/>
        ...
    </application>
</manifest>
```
<center><h6>The default "AppManifest.xml" file from the disassembled package</h6></center>


To make the application available for debugging, you must add the android:debuggable=”true” attribute to the manifest.

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:installLocation="auto" android:targetSandboxVersion="2" package="com.package">
    <uses-permission android:name="android.permission.CAMERA"/>
    ...
    <application android:debuggable="true" android:icon="@mipmap/ic_launcher" android:label="@string/main_app_name" android:largeHeap="true" android:name="com.android.packagename" android:networkSecurityConfig="@xml/network_security_config" android:supportsRtl="false" android:theme="@style/Theme.App">
        <meta-data android:name="tapad.APP_ID" android:value="package-id-tracker"/>
        ...
    </application>
</manifest>
```
<center><h6>The debuggable app manifest</h6></center>

To allow the backup on that app, you must add the android:allowBackup=”true” attribute to the manifest.

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:installLocation="auto" android:targetSandboxVersion="2" package="com.package">
    <uses-permission android:name="android.permission.CAMERA"/>
    ...
    <application android:allowBackup="true" android:debuggable="true" android:icon="@mipmap/ic_launcher" android:label="@string/main_app_name" android:largeHeap="true" android:name="com.android.packagename" android:networkSecurityConfig="@xml/network_security_config" android:supportsRtl="false" android:theme="@style/Theme.App">
        <meta-data android:name="tapad.APP_ID" android:value="package-id-tracker"/>
        ...
    </application>
</manifest>
```
<center><h6>The app manifest is allowing backup of the app now</h6></center>

----

> *I modified everything, what should I do now?*

Rebuild the app and install it on your device looks like a pretty good start. 😄
Apktool is going to be our go-to tools to rebuild the app. 

```bash
> apktool b /path/to/disassemble/apk/ -o  /path/to/disassemble/apk/dbg/pkg.b.apk
```
<center><h6>Rebuild the app using apktool</h6></center>

You can notice that I added “.b” to the extension of the new package.
It will help me to quickly know that this package is the “debuggable” version of the package when I will look for it later.

Now, we must sign the package in order to be able to install on a device.

```bash
> keytool -genkey -v -keystore resign.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
> cp /path/to/disassemble/apk/dbg/pkg.b.apk /path/to/disassemble/apk/dbg/pkg.b.s.apk
> jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore resign.keystore /path/to/disassemble/apk/dbg/pkg.b.s.apk alias_name
```
<center><h6>Sign the package</h6></center>

You can notice that I added “.s” to the extension of the new package. It will help me to quickly that this package is the “debuggable and signed” veresion of the package, ready to be installed on my device.

Uninstall the app that is, at the moment, on your application and reinstall the one you just modified and rebuild. It takes just two adb commands.

```bash
> adb uninstall pkg.name.apk
> adb install /path/to/disassemble/apk/dbg/pkg.b.s.apk
```
<center><h6>Install the new application on the device</h6></center>

Now that you have your application installed on your device, you can start working on it.

Just use the adb shell to connect to your device and start looking for some leaks in the files that are on the device after you used the application.

```bash
> adb shell
$ run-as com.android.packagename
$ com.android.packagename 
```
<center><h6>Connect to your android device and start exploring that local storage</h6></center>

So now, everything is up to you. You must look for something relevant in the device local storage.

I will explain in another post what to look for and some tools that can help you performing offensive security on the application.

----

As you can see, it is interesting to do something like this at the beginning to understand the process. 
But after a few times, that is exhausting to do that same setup again and again.

That’s why I created the following Go program, [androSecTest](https://github.com/Shosta/androSecTest), that does that setup and start performing some attacks, like reverse engineering, insecure local storage, insecure logging and Man in the Middle attacks.

{% github Shosta/androSecTest %}

But more than anything, it creates an application that is available for debugging and install it automatically on the device (and add a nice icon to the app icon so that you can identify quickly which app is available for penetration testing). 
So it brings out all the pain of that setup. And moreover it offers me the opportunity to develop an app using the Go language, which was pretty new to me.

Here is a quick video of this application in action.

{% youtube zzyTFjnwolo %}

Tell me if you find it useful or if you need other features. 
That would mean a lot to me. 

----

*Thanks for taking the time to read this. I hope you were able to learn about the first steps of Offensive Security on an Android device and the benefit of automating.*