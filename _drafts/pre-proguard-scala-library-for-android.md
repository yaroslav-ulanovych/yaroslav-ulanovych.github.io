---
layout: post
title:  "Pre proguard scala library for android"
date:   2014-08-18 11:19:28
categories: scala android proguard maven
---
If you use scala for android development,
you most likely use proguard to work around limitations of dex format.
But shrinking takes quite an amount of build time.
Full build of my project with proguard takes:

| compile  | 10s |
| proguard | 18s |
| dex      | 13s |
| other    | 9s  |
| total    | 50s |

without

| compile | 9s  |
| dex     | 15s |
| other   | 10s |
| total   | 34s |

So removing proguard saves a third of build time.

Instead of a scala library we have to use already proguarded scala library.
And the best place to get it is your own project,
because only it knows what parts of scala library it needs and what it doesn't.
So the algorhythm is: build your application with progard, extract shrunk scala library from it,
replace the origin one, build without proguard.
Enjoy until you face NoSuchMethodError or NoClassDefFoundError
(which means you used scala feature you didn't use before),
repeat that all over again.

That might not be handy, if we couldn't automate the process,
but I managed to set up maven to get the work done and will show you right that.
Let's take a look at the project structure


    cd findnumimg

    # first install the parent pom
    # -N says not to install submodules
    mvn install -N

    # install other modules

    cd core
    mvn install

    cd ../androidapp-deps
    mvn install

    cd ../androidapp
    mvn install -P proguard
    
    # once we have proguarded project
    # we can build our scala library
    cd ../scala-library
    mvn install

    # and build android application without proguard
    cd ../androidapp
    # notice the absence of -P proguard
    mvn package
    mvn android:deploy android:run


{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
