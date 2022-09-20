+++
title = "Jpay_j5_mini"
date = "2022-09-20T08:42:23-07:00"
author = "William Cook"
cover = ""
tags = ["jpay", "incarceration", "jp5 mini"]
keywords = ["incarcerated", "jpay", "hack", "root", "sd card"]
description = "Extracting Content From the JPay JP5 Mini Tablet for Incarcerated Individuals"
showFullContent = false
readingTime = false
hideComments = true
+++

# J5 Mini and Purchased Content
A friend of mine recently finished his sentance at a facility that 
allowed them to purchase the [JP5 Mini](https://www.prnewswire.com/news-releases/jpay-introduces-the-jp5mini-a-highly-customized-android-tablet-for-inmates-300110638.html) tablet for use while incarcerated. After his release, JPay removed the 
in-facility restrictions and allowed them to retain the media they purchased from 
JPay over the course of their sentancing. However, due to the conditions of 
their release, they are unable to operate such electronic devices. They still wanted to keep their purchased media, so I figured I would try my hand. 

## The Problem
The JP5 Mini uses a custom fork of Android 4.2.2 (release date: 2012). 
This custom version of Android contains a lot of security features that 
prevent tampering from inmates. So no opening up the build version and 
enabling developer settings. No ADB capabilities. No mounting for file 
system access over USB. The tablet does have what appears to be a small
storage module soldered to the board where I think the SD card reader 
would be. This prevents inmates from swapping cards and eliminates a 
potential attack vector for hacking. So no just popping it open either. 

## The Solution
Luckily, this version of Android DOES allow for APK installation. APKs 
are Android applications. Think of them like a Windows .exe. There are 
lots of websites that host APK files too for you to download. We need
an APK that runs an FTP server on the tablet. 

### The APK 
The built in broswer is pretty difficult to use given the size and 
resolution the built in screen has. However, it's what we need to use. 
I used [this FTP APK](https://ftp-server-ultimate.apk.gold/android-4.2.2)
(not affiliated, and I make no guarantees about availability or functionality) 
and it worked pretty well. 

### The FTP Server 
FTP is an old protocol that allows files to get shared over a network. 
You can pull files from the remote server or push files to it, 
depending on permissions. We're going to be pulling the purchased media. 
If you're using the APK I linked above, there are some configuration
steps. Make sure the tablet is connected to WiFi and that you know 
the tablet's IP (Settings -> Network, I'm not sure specifically). 

#### Server Configuration
1. Click on the Settings cog in the app.
2. Create new server, give it a name (the name doesn't matter).
3. Click the Users tab at the top of the app. 
4. Create a new user. Both the username and the password are arbitrary
but make sure you remember them because you will need it later. Leave
the Document Root as `/`
5. Save the user and server settings. 
6. Back at the main page of the app, click Start to run the server. 

### The FTP Client
Now that the tablet is running a server, you need to download an FTP 
client on your computer. For Macs, I really enjoy using [CyberDuck](https://cyberduck.io). On Windows, you can use [FileZilla](https://filezilla-project.org). Again, no affiliations or guarantees about these products. Get one 
downloaded and let's configure it together. 

#### Client Configuration
Remember the username and password you set in the server on the tablet
above. You'll need them here. Also get the tablet IP if you didn't before.
1. Connect to a new host. This is the IP from the tablet. The server
should also have generated a random port number. Put that in the Port 
portion of the connection settings. 
2. Enter the username and password from the server settings in their places too. 
3. Click Quick Connect, and you should be given a file tree in the 
connection window. 

#### Getting The Files
The tablet uses that memory module as a virtual SD card, and it's 
accessible a few places. I found it easiest in `/storage/sdcard/`. 
Here are all of the internal media like Music, Pictures, Mail, etc. 
Right click whichever folder you want to download and click the Download
option in the context menu. The network speeds on this device are not
fast, so be patient as they download.

## The Conclusion
This is a pretty weird device without a lot of documentation out on 
the internet. I hope that this guide can show up in more searches when
other folks go digging for information on the JP5 Mini. 