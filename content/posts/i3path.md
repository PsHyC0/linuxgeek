---
title: "I3WM path"
date: 2023-08-04T11:51:44-07:00
tags: [ "linux","i3wm","desktop" ]
draft: false
image: img/i3wm.png
---
Running i3wm and found that with rofi it would not see executables I put in $HOME/bin directory.

The problem is that the $PATH variable is set correctly for command line items but not within the GUI/WM. Googled solutions and people said to put things under /usr/local/bin, etc. 

I want personal sripts and crap under $HOME/bin.

To see the $PATH that i3 recognizes: 

```bash
$ cat /proc/$(pidof i3)/environ | tr '\0' '\n' | grep '^PATH'
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/lib/jvm/default/bin:/usr/bin/site_perl:/usr/bin/vendor_perl:/usr/bin/core_perl
```
Notice no /home/$USERNAME/bin in the PATH

I tried putting the export statement into the .xsession file and other iterations of the file with no luck.

Solution was to add line to ~/.profile file

```bash
$ echo 'export PATH=$HOME/bin:$PATH' >> ~/.profile
```

Log out and back in again and things should be working now.



