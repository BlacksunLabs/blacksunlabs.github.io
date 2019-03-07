---
author: noncetonic
layout: post
title: macOS Persistence via iTerm
---

## Foreword ##
macOS's default `Terminal.app` is garbage, and everyone knows it; they also know iTerm is by and large the most popular terminal replacement for macOS. But did you know iTerm is also a fantastic way to persist on a macOS host?

I've been sitting on this for far too long and quite honestly I had completely forgotten I had never shared this fun little technique. So here it is.

## Background ##
There are two directories which iTerm2 checks for `AutoLaunch` scripts during execution. These AutoLaunch scripts are intended to allow users to automate a number of start-up tasks via iTerm2's AppleScript Scripting Definition. While this is the intention, iTerm2 will blindly execute whatever code is in the `Autolaunch.scpt` file regardless of whether or not the code bothers to interact with iTerm2.

Autolaunch.scpt locations:

- `~/Library/Application Support/iTerm2/Scripts/AutoLaunch.scpt` 
	- This location is checked first and may not exist on your system
	- Go ahead and create the `~/Library/Application Support/iTerm2/Scripts/` directory if it doesn't already exist.
	- If this directory doesn't exist the next path is checked as a legacy fallback.
- `~/Library/Application Support/iTerm/Scripts/AutoLaunch.scpt`
	- !! DEPRECATION NOTICE !! This path is only being checked due to legacy reasons and should be considered deprecated. If you rely on this path to always be an option it may eventually come back to bite you.

> Note: This only kicks off during application launch, spawning an extra tab in your terminal won't do much and neither will spawning a new window. 

## PoC||GTFO ##
This one is pretty easy. Honestly it's almost too easy which makes me wonder why I haven't heard of other people using this technique. 

### BASHing buttons ###
Need to show this off to your security team and don't want to invest a whole lot of time on a PoC? This one is for you.

```
#!/bin/bash
#
# bsrl_iTerm-00.sh
# Useless payload but it gets the point across
#
##############################################
say "iTerm, uTerm, we all Term for code exec"
```

Go ahead and toss that into `~/Library/Application Support/iTerm2/Scripts/AutoLaunch.scpt` ( please make sure you name is AutoLaunch.scpt :3 ), unmute your speakers, and restart iTerm2.

What's great is that this will run regardless of whether or not it has executable permissions set. Perfect.

### Wait whut? ###
At first I was pretty excited having instantly assumed the file extension made no difference to iTerm. That would've been nice.

I honestly haven't had enough spare fscks to dig into this in order to be certain but I tested with a simple python script to no avail.

When I sat back I realized the reason the bash script worked was due to the fact that `say` is part of the AppleScript language and all the other lines are comments... 

Whoops!

### Snakes on a plane ###
Not all is lost if you don't want to write AppleScript though. Luckily AppleScript allows us to run shell commands which opens up a whole host of built-in languages for us.

```
(*
 * bsrl_iterm-01.scpt
 *
 * Downloads a random image from XKCD
 * and opens it with whichever app is
 * registered as its opener 
 * (Preview.app by default)
 *)
set theCommand to quoted form of "import urllib;urllib.urlretrieve(\"https://imgs.xkcd.com/comics/random_number.png\", \"/tmp/xkcd.png\")"
do shell script "/usr/local/bin/python " & "-c " & theCommand
do shell script "/usr/bin/open " & "/tmp/xkcd.png"
```

Kinda cool? Not really...

### But wait; there's more! ###
What if I told you that in addition to learning something fun about persistence, today you were also blessed with the gift of command execution within a real life shell :O. Just think of all the fun you could have with the ability to wait for a user to run `sudo` or `ssh` to a server and then inject your dirty bits right into their console.

This script will work as an `AutoLaunch.scpt` but it can also be run at any time to inject keystrokes into any window, any session, and any tab.


```
tell application "iTerm"
	tell current session of current window
		set thePath to quoted form of "/tmp/bsrl_iterm-02.txt"
		write text "echo " & "'yay RCE'" & " >> " & thePath
		write text "say we did it"
	end tell
end tell
```

>>> *Note* On macOS 10.14, Mojave has locked down the otherwise wide open Apple Events system and now requires that applications be given express permission to send Apple Events to other applications. Because of this security enhancement it is currently not possible to inject commands into the context of iTerm without prompting the user for permission –unless they have previously allowed it. 

>>>The above technique will allow you a small window for injecting commands into iTerm. Ambitious readers can surely devise schemes which take advantage of this window for more nefarious deeds. If you wind up writing something and would like to share or discuss it, feel free to reach out to me via twitter @noncetonic

## Closing ##
Welp that's all for now folks. Have fun with the new toys in your arsenal.

### Further Reading ###
For those of you who are interested in learning more about some of the topics I glossed over in this post I've included some links.

- iTerm2's Scripting Documentation : https://iterm2.com/documentation-scripting.html
- Everyday AppleScriptObjC 3rd Edition : https://sites.fastspring.com/myriadcommunications/product/everydayasobjc 
- AppleScript Language Guide : https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/introduction/ASLR_intro.html#//apple_ref/doc/uid/TP40000983-CH208-SW1
