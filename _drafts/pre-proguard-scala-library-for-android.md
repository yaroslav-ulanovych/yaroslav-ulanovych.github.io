---
layout: post
title:  "Pre proguard scala library for android"
date:   2014-08-18 11:19:28
categories: scala android proguard maven
---
If you use scala for android development,
you most likely need proguard to work around limitations of dex format.
But shrinking takes quite an amount of build time, for instance for my project
I got these numbers:

with shrinking

| compile  | 10s |
| proguard | 18s |
| dex      | 13s |
| other    | 9s  |
| total    | 50s |

without shrinking

| compile | 9s  |
| dex     | 15s |
| other   | 10s |
| total   | 34s |

So removing proguard saves a third of a build time. It is absolutely not
nececcary to perform it each build. Having apk built with proguard you may 
extract scala library from it and use instead of the origin one without
proguard. Enjoy until you face NoSuchMethodError or NoClassDefFoundError
(which means you used scala feature you didn't use before), just repeat that
all over again. That might be not handy, if we couldn't automate the process,
but I managed to set up maven to get the work done and will show you right that.
There is an example project where you can see all the details

    git clone https://github.com/yaroslav-ulanovych/findnumimg
    cd findnumimg

I created a tag specially for this article, cause the project may change
in future

    git checkout for-pre-proguard-scala-library-for-android-blog-post

Build instructions

    cd findnumimg

    # first install the parent pom
    # -N says not to install submodules
    mvn install -N

    # install other modules

    cd core
    mvn install

    cd ../androidapp-deps
    mvn install

    # build android app with proguard
    cd ../androidapp
    mvn install -P proguard

    # check that it works
    mvn android:deploy android:run
    
    # once we have proguarded apk
    # we can build our scala library
    cd ../scala-library
    mvn install

    # and build android app without proguard
    cd ../androidapp
    # notice the absence of -P proguard
    mvn package

    # check that it works
    mvn android:deploy android:run

Now let's discuss maven configuration in details. There is nothing interesting
about parent project, it's ordinary, just common settings for submodules.
Core project contains application code that is not android specific.
In fact you can put it's code in androidapp project and get rid of core and
parent projects, those are just my enterprise habits. So real magic happens in
androidapp, androidapp-deps and scala-library modules (and if maven was a bit
smarter, we could get rid of andoidapp-deps module too).

# scala library module

Module produces a jar artifact which contains shrunk scala library. That is
just five console commands but in maven it's a hundred of lines in xml
(still like maven).

[Tool][dex2jar] that I use to convert dex to jar names differently in Windows
and Linux, fixing that.

        <profiles>
            <profile>
                <id>Windows</id>
                <activation><os><family>Windows</family></os></activation>
                <properties><dex2jarExecutable>d2j-dex2jar.bat</dex2jarExecutable></properties>
            </profile>
            <profile>
                <id>Linux</id>
                <activation><os><family>Linux</family></os></activation>
                <properties><dex2jarExecutable>d2j-dex2jar.sh</dex2jarExecutable></properties>
            </profile>
        </profiles>

Don't pay attention at phases plugins are bound to, I had problems running the
same plugin twice in one phase in correct order, so just put them in different
phases.

First, copy proguarded apk from local repository to build directory

                    ...
                    <artifactId>maven-dependency-plugin</artifactId>
                    ...
                            <id>copy apk</id>
                    ...
                                    <artifactItem>
                                        <groupId>com.mahpella.findnumimg</groupId>
                                        <artifactId>androidapp</artifactId>
                                        <version>${version}</version>
                                        <classifier>proguard</classifier>
                                        <type>apk</type>
                                        <overWrite>true</overWrite>
                                        <outputDirectory>${project.build.directory}</outputDirectory>
                                        <destFileName>androidapp.apk</destFileName>
                                    </artifactItem>
                    ...

Unzip it


                    ...
                    <artifactId>maven-antrun-plugin</artifactId>
                    ...
                                    <unzip src="${project.build.directory}/androidapp.apk" dest="${project.build.directory}/apk" />
                    ...

Convert classes.dex to jar via [dex2jar][dex2jar] (should be installed and
available in path)

                    ...
                    <artifactId>exec-maven-plugin</artifactId>
                    ...
                            <id>dex2jar</id>
                    ...
                                    <argument>${project.build.directory}/apk/classes.dex</argument>
                                    <argument>-o</argument>
                                    <argument>${project.build.directory}/classes.jar</argument>
                    ...

Unzip

                    ...
                    <artifactId>maven-antrun-plugin</artifactId>
                    ...
                            <id>unzip classes.jar</id>
                    ...
                                    <unzip src="${project.build.directory}/classes.jar" dest="${project.build.outputDirectory}"/>
                    ...

Remove everything but scala library classes.

                    ...
                    <artifactId>exec-maven-plugin</artifactId>
                    ...
                            <id>remove not scala-library classes</id>
                    ...
                                <executable>rm</executable>
                                <arguments>
                                    <argument>-rf</argument>
                                    <argument>${project.build.outputDirectory}/com</argument>
                                    <argument>${project.build.outputDirectory}/android</argument>
                    ...

Maven jar plugin will do the rest.

# androidapp module

By default it should depend on shrunk scala library and via proguard profile
on the origin one. But the problem with that is that in maven we can't just
[exclude dependencies in profile][cantdeactivatedependencies], but we can do
that with transitive ones and that is what androidapp-deps module is for. It
holds dependencies of the androidapp module so that we can exlude shrunk scala
library in proguard profile

            ...
            <dependency>
                <groupId>com.mahpella.findnumimg</groupId>
                <artifactId>androidapp-deps</artifactId>
                <version>1.0-SNAPSHOT</version>
                <type>pom</type>
                <exclusions>
                    <exclusion>
                        <groupId>org.scala-lang</groupId>
                        <artifactId>scala-library</artifactId>
                    </exclusion>
            ...

Parser combinators are a separate module, not part of scala library but I
include them in shrunk scala library

            ...
                    <exclusion>
                        <groupId>org.scala-lang.modules</groupId>
                        <artifactId>scala-parser-combinators_${scalaVersion}</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            ...

Exclude shrunk scala library and include orignal one in proguard profile

        ...
        <profiles>
            <profile>
                <id>proguard</id>
                <dependencies>
                    <dependency>
                        <groupId>org.scala-lang</groupId>
                        <artifactId>scala-library</artifactId>
                        <version>${scalaVersionFull}</version>
                    </dependency>
                    <dependency>
                        <groupId>com.mahpella.findnumimg</groupId>
                        <artifactId>androidapp-deps</artifactId>
                        <version>1.0-SNAPSHOT</version>
                        <type>pom</type>
                        <exclusions>
                            <exclusion>
                                <groupId>com.mahpella.findnumimg</groupId>
                                <artifactId>scala-library</artifactId>
                            </exclusion>
                        </exclusions>
        ...

Enable proguard

        ...
                                <proguard><skip>false</skip></proguard>
        ...

Give proguarded apk a classifier to distinct in from a full one

        ...
                                <classifier>proguard</classifier>
        ...




{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}


[dex2jar]: https://code.google.com/p/dex2jar
[cantdeactivatedependencies]: http://stackoverflow.com/a/1790230/1351319