---
layout: post
title: Downgrading Broken Packages
Description: Downgrading Broken Packages
date: 2021-10-12 00:59:35  +0300
tags: [arch-linux,package-managers]
image: '/assets/img/posts/downgrade/dgbanner.png'
featured: false
---

It is a small possibility, but not impossible, that you may have encountered
with a broken package after updating your system, while using rolling release
operating system. Packages pushed into the repositories without going under
any superior control. For example, when new version of a package arrives,
manjaro checks it in it's containers, then push them into public repos. While
it is quite a personal preference which approach is better for you, it is not the
same for each operating system. So if you have a broken package, or you are not just
not comfortable with newer version of the release you may wish to downgrade it. 
In this post you will learn how to downgrade a package and ignore updates.

Download utility called **downgrade** with your favorite package manager.
{% highlight bash %}
yay -S downgrade
{% endhighlight %}
After installing it run
{% highlight bash %}
sudo downgrade cmatrix
{% endhighlight %}
It will prompt you to a selection screen between version just like that
{% highlight bash %}
Available packages (community):

   1)  cmatrix    2.0  1  remote
+  2)  cmatrix    2.0  2  remote
+  3)  cmatrix    2.0  2  /var/cache/pacman/pkg

select a package by number: 
{% endhighlight %}
Select which version of the program you wish to use than press enter.
After that step, downgrade utility will ask you to whether you wish to
add that program to ignorepackage list to prevent later updates or not.
{% highlight bash %}
add cmatrix to IgnorePkg? [y/N]
{% endhighlight %}
If you press **y** in this step, the package wont be affected while doing
system wide updates. However if you wish to change it back and update the
software someday, you need to edit **/etc/pacman.conf**. Go find **IgnorePkg=**
variable, and delete the package name there. You can also add other packages 
here manually to disable updates for a spesific program.
