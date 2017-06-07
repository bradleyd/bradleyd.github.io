---
layout: post
title: "Use Systemd to control virtual machines"
date: 2017-06-06
---

![robot](/images/robot.png){:height="175px" width="150px"}

My company is building an on premise solution for one of our customers.  We chose to use virtual machines via KVM and [libvirt](https://libvirt.org/).  We had an issue where we needed to the guests to come up in order if the host restarted.  We also needed a way to restart guests if they crashed or they get stopped.  There are tons of ways to skin this metal clone, but after some tinkering we decided to leverage [systemd](https://www.freedesktop.org/wiki/Software/systemd/)

Here is service file we use to make this work

```bash
[Unit]
Description=Foo Guest
After=bar-guest.target

[Service]
Type=forking
ExecStart=/usr/bin/virsh start foo
ExecStop=/usr/bin/virsh stop foo

[Install]
WantedBy=multi-user.target
```

