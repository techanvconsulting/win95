---
title: Release and Renew IP in OS X
author: Chris Titus

date: 2011-09-29T18:46:43+00:00
url: /release-and-renew-ip-in-os-x/
categories:
  - MacOS

---
I recently had some issues where a bridged virtual machine in osx would disconnect my osx side network connection. A simple release and renew would fix this. <!--more-->Simply open up Terminal in OSX and type:

`sudo ifconfig en0 down`
  
`sudo ifconfig en0 up`

Obviously if you are renewing another interface other than your Ethernet port, en0 would need to be changed to the corresponding short name of the interface.

