---
layout: post
title: Add Windows Boot Manager to Grub
date: 2021-11-18 12:00
tags: [linux, windows, grub, boot-managers]
image: '/assets/img/posts/add-windows-boot-manager-to-grub/wgbanner.png'
featured: false
---
If you want to add windows boot manager to grub, follow these steps;

1-Download **os-prober**<br>
2-Run:
{% highlight bash %}
sudo os-prober
{% endhighlight %}
If there is an output like that, you are good to go:
{% highlight bash %}
-> sudo os-prober
/dev/nvme1n1p1@/EFI/Microsoft/Boot/bootmgfw.efi:Windows Boot Manager:Windows:efi
{% endhighlight %}
<br>
3- Append this line, **GRUB_DISABLE_OS_PROBER=false**, to the **/etc/default/grub** file.<br>
4- Run: 
{% highlight bash %}
sudo grub-mkconfig -o /boot/grub/grub.cfg
{% endhighlight %}
<br>
Done ! You will see all bootable system in your grub menu at the next boot.
