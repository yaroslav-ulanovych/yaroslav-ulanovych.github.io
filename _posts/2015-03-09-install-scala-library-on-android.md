---
layout: post
title:  "Install scala library on Android"
tags: scala android proguard
author: Yaroslav Ulanovych
permalink: install-scala-library-on-android.html
---

I'm still struggling with scala, android, proguard
and the ridiculous amount of time to build an android scala application.
My [previous attempt]({% post_url 2014-09-02-pre-proguard-scala-library-for-android-with-maven %})
works but it sucks, cause every time you use a new function from scala library you:

1. Build and launch the application.
2. Click to the new feature you implemented.
3. See an exception in logs and realize that it occured because of an absent scala method or class.
4. Build the application with proguard and full scala library.
5. Make shrunk scala library.

That's annoying and takes some time. So I put some effort looking for another solution.
And I found two! One of them didn't work for me though, but second did and that's what I'm going to tell you.

The idea of both solutions is to install scala library to the android phone you use for development,
so that you don't need to pack it with the application. That's obviously not a way for release builds,
cause users won't have a scala library, but we don't do releases often so can live with proguard for that.

The [first way](http://zegoggl.es/2011/07/how-to-preinstall-scala-on-your-android-phone.html)
is to include scala-library to `BOOTCLASSPATH`. This is an environment variable similar to java classpath,
it's set in the `init.rc` script. The thing is that we can't just go and edit that script,
we need to get the boot image, unpack it, fix the script, pack the image, flash it back to the device.
And that's when the hell comes in with vendor specific differences, os dependent tools, buggy software,
pieces of information scattered by forums. The tool I used for repack the kernel seemed to work fine,
it printed image info, extracted kernel but failed to extract ramdisk saying my ramdisk isn't gzip.
I thought that I got bad boot image, but on some forum I read that this tool (written in Perl)
is not tested on Windows and indeed on Linux it worked, but to flash the boot image I had to use Windows tools.
I spent almost two days and in the end my phone didn't boot, it was showing vendor logo, I could `adb shell` in it
(but not `su`), there was some stuff going on in `adb logcat`, it lasted for an half an hour,
then I pull out the battery and restored original boot image.
Maybe my poor cheap phone with 512MB RAM couldn't handle all that stuff.

The way that worked is
[`uses-library`](http://developer.android.com/guide/topics/manifest/uses-library-element.html).
This android manifest element specifies a shared library that the application must be linked against.
Not a `*.so` library, a java jar library.

Note, the instructions below work for me and my Lenovo A630 phone based on MediaTek MT6577 chip
with some custom rooted Android 4.0.4, I do not claim that this solution works for every device.

First we need to convert scala library to dex fomat
{% highlight bash %}
java -jar $ANDROID_HOME/build-tools/21.1.2/lib/dx.jar --dex --output=scala-library-2.11.5-dex.jar scala-library-2.11.5.jar
{% endhighlight %}

and create permissions file `scala-library-2.11.5-dex.xml` with the following content

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<permissions>
    <library name="scala-library-2.11.5-dex" file="/system/framework/scala-library-2.11.5-dex.jar" />
</permissions>
{% endhighlight %}

{% highlight bash %}
# push jar and permissions files somewhere to the phone
adb push scala-library-2.11.5-dex.jar /mnt/sdcard
adb push scala-library-2.11.5-dex.xml /mnt/sdcard

# enter shell
adb shell

# go root
su

# remount the system partition in the read-write mode
mount -o rw,remount /system

# put the jar to the proper place
cd /system/framework
cp /mnt/sdcard/scala-library-2.11.5-dex.jar ./
chmod 644 scala-library-2.11.5-dex.jar
rm /mnt/sdcard/scala-library-2.11.5-dex.jar

# put the permissions to the proper place
cd /system/etc/permissions
cp /mnt/sdcard/scala-library-2.11.5-dex.xml ./
chmod 644 scala-library-2.11.5-dex.xml
rm /mnt/sdcard/scala-library-2.11.5-dex.xml

reboot
{% endhighlight %}

The last things to do is to add the scala library to the application element of the manifest

{% highlight xml %}
<application android:label="@string/app_name" android:icon="@drawable/icon">
    <uses-library android:name="scala-library-2.11.5-dex" android:required="true"/>
{% endhighlight %}

turn scala library dependency to the provided scope

{% highlight xml %}
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>${scalaVersionFull}</version>
    <scope>provided</scope>
</dependency>
{% endhighlight %}

and enjoy.