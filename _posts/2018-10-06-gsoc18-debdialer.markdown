---
title: "DebDialer : Handling phone numbers on Linux Desktops | GSoC 2018"
layout: post
date: 2018-10-06 07:34
tag: 
- intro
- gsoc
- python
image: ../assets/images/deb.png
headerImage: true
projects: true
hidden: false 
description: "GSoC 2018 project under Debian : An Python application to handle phone numbers and :tel URIs on a Linux Desktop"
category: blog
author: Vishal Gupta
externalLink: false
---
This summer I had the chance to contribute to Debian as a part of GSoC. I built a desktop application, debdialer for handling tel: URLs and (phone numbers in general) on the Linux Desktop. It is written in [Python 3.5.2](https://www.python.org/downloads/release/python-352/) and uses [PyQt4](http://pyqt.sourceforge.net/Docs/PyQt4/introduction.html#pyqt4-components) to display a popup window. Alternatively, there is also a `no-gui` option that uses [dmenu](https://wiki.archlinux.org/index.php/Dmenu) for input and terminal for output. There is also a [modified apk](tiny.cc/ddial-kdeconnect) of [KDE-Connect](https://phabricator.kde.org/project/view/159/) to link debdialer with the user's Android Phone. The pop-up window has numeric and delete buttons, so the user can either use the GUI or keyboard to modify numbers.

<div class="side-by-side">
    <div class="toleft">
        <img class="image" src="http://vishalgupta.me/debdialer/Images/PrimaryDesk.png" alt="Alt Text">
        <figcaption class="caption">DebDialer</figcaption>
    </div>

    <div class="toright">
        <p><h4> Features</h4>(<a href = "https://salsa.debian.org/comfortablydumb-guest/Hello-from-the-Debian-side#usage">Screenshots and how-to</a>)
        <ol>
            <li> Adds contact using .vcf file (<i>Add vcard to     Contacts</i>)</li>
            <li> Adds number in dialer as contact (<i>Add to Contacts</i>) </li>
            <li> Sending dialer number to Android phone (<i>DIAL ON ANDROID PHONE</i>)</li>
            <li> Parsing numbers from file (<i>Open File</i>) </li>
            <li> Automatic formatting of numbers and setting of details </li>
        </ol>
        </p>
    </div>
</div>

## Installation
Installing with `pip` installs the python package but does not set up the desktop file. Hence, the following script needs to be run.
```
# Optional dependencies. Atleast one of them is required.
sudo apt install dmenu
sudo apt install python3-pyqt4

curl -L https://salsa.debian.org/comfortablydumb-guest/Hello-from-the-Debian-side/raw/master/install.sh -s | bash
```
## Links :
- debdialer
  - [Debian Salsa Repository ](https://salsa.debian.org/comfortablydumb-guest/Hello-from-the-Debian-side/tree/master)
  - [Github Mirror](https://github.com/py-ranoid/debdialer)
   - Email : [vishalg8897@gmail.com](mailto:vishalg8897@gmail.com)