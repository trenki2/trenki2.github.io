---
layout: post
title:  "Trading on Raspberry Pi with Ubuntu 18.04"
date:   2019-05-14 14:15:00 +0200
last_modified_at:   2019-08-21 12:50:00 +0200
feature_image: "https://picsum.photos/id/705/1300/400"
category: Trading
tags: [stock, trading, trading automation, backtesting]
---

I wrote my own backtesting and live trading software called ArgonTrader in C#
and use it to trade with Interactive Brokers. I run it on my Raspberry Pi 3 B. In
this post I describe how to set things up using **Ubuntu MATE 18.04**.

<!-- more -->

## Install Ubuntu on the Raspberry Pi

I used the 32 bit Raspberry Pi 3 (Hard-Float) preinstalled server image from
[here](http://cdimage.ubuntu.com/releases/18.04/release/). This image was the
only one that allowed me to install `raspi-config` and also Oracle Java 8.

You can put the image on a SDHC card, connect keyboard, mouse and monitor to the
Raspberry Pi and boot. You can then login with the username and password
"ubuntu" and change the password afterwards.

To install the Ubuntu MATE desktop you can use the following commands.

```bash
sudo apt install tasksel
sudo apt update
sudo tasksel install ubuntu-mate-desktop
```

## Raspi-Config

You will have to install `raspi-config` yourself as it is not included.

To install it use these commands:

```bash
sudo echo "deb http://archive.raspberrypi.org/debian/ jessie main" >> /etc/apt/sources.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 7FA3303E
sudo apt-get update
sudo apt-get install raspi-config
```

## SSH

To enable SSH you can use `raspi-config`. 

```bash
sudo mount /dev/mmcblk0p1 /boot     # Required when you have the Ubuntu server version
sudo raspi-config
```

Once you have started `raspi-config` you can go to *Interface options* and enable SSH.

## VNC Server

For the VNC server i did not use `raspi-config` but installed the
`tightvncserver` as described in this [blog
post](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-18-04).

The `tightvncserver` can be set up as a service that persists, so you can connect
and disconnect from the server and the desktop session stays alive.

## Timezone

For setting up the timezone you can also use `raspi-config`.

## Wifi

If you want to use Wifi with your Raspberry Pi without having to login you
probably have to set this up, when it does not work out of the box.

You have to create a `netplan` configuration file:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

And put this content:

```text
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: yes
      access-points:
        "YOUR_ACCESS_POINT_NAME":
          password: "YOUR_PASSWORD"
```

You should check `iwconfig` to make sure `wlan0` is your wifi device, otherwise
you have to change this to the correct value in the config.

After saving the configuration file you should run the following commands:

```bash
sudo netplan generate
sudo netplan apply
```

## Mono

My own trading software is written in C# and under Linux I need Mono to run it.
I followed the instructions to install the most up to date Mono from their
[homepage](https://www.mono-project.com/download/stable/#download-lin):

```bash
sudo apt install gnupg ca-certificates
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
sudo apt update

sudo apt-get install mono-complete
```

## Oracle Java

```bash
sudo apt install oracle-java8-jdk
```

You can then check the installed version with

```bash
java -version
```

## IB Gateway

Installing the IB Gateway is a bit tricky. The default setup for linux includes
its own JVM and does not support ARM. It cannot be installed on the Raspberry
Pi without modification.

Download the IB Gateway software from Interactive brokers and open it in vim in
binary mode:

```bash
vim -b ibgateway-stable-standalone-linux-x64.sh
```

At the beginning there is a line with `INSTALL4J_JAVA_HOME_OVERRIDE`. I
uncommented it and pointed it to the installed Java Virtual Machine.

```bash
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt
```

A bit further down in the script there is a `test_jvm()` function which does a
version check. Modifying the version numbers that are checked for, so that the
installed jvm will be accepted, allows one to run the script.

More precisely there is the following code, that needs to be changed. Just set
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