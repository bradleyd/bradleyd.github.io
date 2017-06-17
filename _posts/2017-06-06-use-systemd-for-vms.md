---
layout: post
title: "Use Systemd to control virtual machines"
date: 2017-06-06
---

![robot](/images/robot.png){:height="175px" width="150px"}

My company is building an on premise solution for one of our customers.  We chose to use virtual machines via KVM and [libvirt](https://libvirt.org/).  We had an issue where we needed the guests to come up in a specific order if the host restarted.  We also needed a way to monitor the guests if they crashed or got stopped.  There are tons of ways to skin this metal cat, but after some tinkering we decided to leverage [systemd](https://www.freedesktop.org/wiki/Software/systemd/)

Here is a service file we use to make this work.

```bash
[Unit]
Description=Auth Guest
After=rabbit-guest.service

[Service]
Type=forking
PIDFile=/var/run/libvirt/qemu/auth.pid
User=root
ExecStart=/usr/bin/virsh start auth
ExecStop=/usr/bin/virsh destroy auth
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Here we have our `auth` guest which can only be started after the rabbitmq guest is up and running.  The key here is the `forking` type and the `pid file`.  The guest is also configured to always restart.

During our testing, this process proved very reliable.  One caveat though, let systemd control the lifecycle of the guest, do not use `virsh` start stop commands mixed with `systemctl` commands.
