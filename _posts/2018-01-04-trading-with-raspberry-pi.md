---
layout: post
title:  "Trading with Raspberry Pi"
date:   2018-01-04 16:00:00 +0200
feature_image: "https://unsplash.it/1300/400?image=1057"
category: Trading
tags: [stock, trading, trading automation, backtesting]
---

I wrote my own backtesting and live trading software called ArgonTrader in C#
and use it to trade with Interactive Brokers.
I run it on my Raspberry Pi. In this post I describe my setup.

<!-- more -->

## Ubuntu MATE on the Pi

I like Ubuntu and that is why I chose to install Ubuntu MATE on the Raspberry
Pi. I prefer it over the default Raspbian. Ubuntu MATE can be downloaded
[here](https://ubuntu-mate.org/raspberry-pi/).

I wrote the disk image onto my micro SDHC card, inserted it into the Pi and
followed the setup instructions. The setup worked flawlessly and afterwards 
I had a working Ubuntu.

I used `raspi-config` to change some default configurations settings and enabled
**ssh** and set the screen resolution.

## VNC Server

I installed the **tightvncserver** vnc server package in Ubuntu so that I am
able to log in remotely.

```bash
sudo apt install tightvncserver
```

After installing the server I followed this
[tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-16-04)
to setup the vnc server as a service and also configure the screen resolution.

Open the service file:

```bash
sudo nano /etc/systemd/system/vncserver@.service
```

Insert this content and replace the user name and change the resolution.

```txt
/etc/systemd/system/vncserver@.service 
[Unit]
Description=Start TightVNC server at startup
After=syslog.target network.target

[Service]
Type=forking
User=sammy
PAMName=login
PIDFile=/home/sammy/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver -depth 24 -geometry 1280x800 :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```

I enabled the vnc server service with the following command and now I am able to
connect with a VNC Viewer application from my desktop machine to Ubuntu.

```bash
sudo systemctl start vncserver@1
```

## Mono

My own trading software is written in C# and under Linux I need Mono to run it.
I followed the instructions to install the most up to date Mono from their homepage:

```bash
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
echo "deb http://download.mono-project.com/repo/ubuntu xenial main" | sudo tee /etc/apt/sources.list.d/mono-official.list
sudo apt-get update
```

## Oracle Java

For running the IB Gateway trading software I needed the Oracle JDK. This can be
installed with the following commands:

```bash
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

## IB Gateway

Installing the IB Gateway is a bit tricky. The default setup for linux includes
its own JVM and does not support ARM. It cannot be installed on the Raspberry
Pi.

There is a IB Gateway Standalone version. It is possible to run this one on the
Raspberry Pi but unfortunately in my tests it did not work well with the current
API 9.72 and had errors when I was trying to make trades.

I wanted the most current stable version of IB Gateway on the Raspberry Pi. I
solved the problem by installing Ubuntu x64 on a virtual machine with the same
username as the Raspberry Pi and then installing the current stable IB Gateway.
I then copied the installed files containing the jar files to the Raspberry Pi
and modified the startup script.

At the beginning of the startup script there is a line that accepts a path to an
alternate JVM to use:

```bash
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/java-8-oracle/
```
The script also contains code to test the jvm in `test_jvm()` at a specified
location. The script does a version check which checks for a specific jvm
version. Since the one from the ppa repository is not exactly the same as the
one checked for in the script, I removed the checks with comments.

## Conclusion

After all these steps I am able to run my trading bot on the Raspberry Pi. The
Raspberry Pi is fast enough for my purposes, I trade only at the end of the day.
Installing the IB Gateway is a bit tricky but doable.