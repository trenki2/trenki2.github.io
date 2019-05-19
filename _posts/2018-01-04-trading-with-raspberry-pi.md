---
layout: post
title:  "Trading on Raspberry Pi with Ubuntu 16.04"
date:   2018-01-04 16:00:00 +0200
last_modified_at:   2019-05-19 10:00:00 +0200
feature_image: "https://unsplash.it/1300/400?image=1057"
category: Trading
tags: [stock, trading, trading automation, backtesting]
---

I wrote my own backtesting and live trading software called ArgonTrader in C#
and use it to trade with Interactive Brokers.
I run it on my Raspberry Pi. In this post I describe my setup.

<!-- more -->

This post uses **Ubuntu 16.04** as a base. If you want to use **Ubuntu 18.04**
instead you should check out my [other post]({% post_url
2019-05-14-trading-with-raspberry-pi %}).

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

sudo apt-get install mono-complete
```

## Oracle Java

For running the IB Gateway trading software I needed the Oracle JDK. This can be
installed with the following commands:

```bash
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

You can then check the installed version with

```bash
java -version
```

## IB Gateway

Installing the IB Gateway is a bit tricky. The default setup for linux includes
its own JVM and does not support ARM. It cannot be installed on the Raspberry
Pi without modification.

There is a IB Gateway Standalone version. It is possible to run this one on the
Raspberry Pi but unfortunately in my tests it did not work well with the current
API 9.72 and had errors when I was trying to make trades.

I wanted the most current stable version of IB Gateway on the Raspberry Pi.
To make the installer work on ARM it is possible to edit it with vim in binary mode:

```bash
vim -b ibgateway-stable-standalone-linux-x64.sh
```

At the beginning there is a line with `INSTALL4J_JAVA_HOME_OVERRIDE`. I
uncommented it and pointed it to the installed Java Virtual Machine.

```bash
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/java-8-oracle/
```

A bit further down in the script there is a `test_jvm()` function which does a
version check. Modifying the version numbers that are checked for, so that the
installed jvm will be accepted, allows one to run the script.

More precicely there is the following code, that needs to be changed. Just set
the version to the one you have.

```
136   if [ "$ver_major" = "" ]; then
137     return;
138   fi
139   if [ "$ver_major" -lt "1" ]; then
140     return;
141   elif [ "$ver_major" -eq "1" ]; then
142     if [ "$ver_minor" -lt "8" ]; then
143       return;
144     elif [ "$ver_minor" -eq "8" ]; then
145       if [ "$ver_micro" -lt "0" ]; then
146         return;
147       elif [ "$ver_micro" -eq "0" ]; then
148         if [ "$ver_patch" -lt "152" ]; then
149           return;
150         fi
151       fi
152     fi
153   fi
154 
155   if [ "$ver_major" = "" ]; then
156     return;
157   fi
158   if [ "$ver_major" -gt "1" ]; then
159     return;
160   elif [ "$ver_major" -eq "1" ]; then
161     if [ "$ver_minor" -gt "8" ]; then
162       return;
163     elif [ "$ver_minor" -eq "8" ]; then
164       if [ "$ver_micro" -gt "0" ]; then
165         return;
166       elif [ "$ver_micro" -eq "0" ]; then
167         if [ "$ver_patch" -gt "152" ]; then
168           return;
169         fi
170       fi
171     fi
172   fi
```

**After successful installation the same modifications have to be done to the
launcher script for IB Gateway.**

## Conclusion

After all these steps I am able to run my trading bot on the Raspberry Pi. The
Raspberry Pi is fast enough for my purposes, I trade only at the end of the day.
Installing the IB Gateway is a bit tricky but doable.