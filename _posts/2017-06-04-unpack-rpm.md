---
layout: post
title: "Un-pack a RPM"
date: 2017-05-04 18:12:00
---

I recently had to create a RPM from a DEB package.  I used [alien](https://wiki.debian.org/Alien). I tried to install it and there was some errors.  I needed to see what was in the archive.

~~~bash
rpm2cpio ./example.rpm | cpio -idmv
~~~

This command will extract the RPM in the directory from wherever this command is run.

