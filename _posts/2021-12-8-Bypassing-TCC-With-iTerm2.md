---
author: actae0n 
layout: post
title: Bypassing TCC With iTerm2
tags: [MacOS, Post-Exploitation]
---

# Motivation
When landing a shell in MacOS environments, you'll frequently want access to the files under your victim's home folder. Some of these locations are TCC protected. Specifically  `~/Documents/`, `~/Downloads/`, `~/Desktop/` all require their own approvals which correspond to the `kTCCServiceSystemPolicyDocumentsFolder`, `kTCCServiceSystemPolicyDownloadsFolder`, `kTCCServiceSystemPolicyDesktopFolder` service categories respectively. Depending on how you landed your initial access, you may not have the necessary TCC approvals. If your target has iTerm2 installed and they use it for day-to-day work, you may have a simple way around these TCC restrictions. 

# iTerm2 AutoLaunch Scripts
iTerm2 has support for autolaunching an AppleScript file at startup, if it's placed at a predetermined path, as documented [here](https://iterm2.com/documentation-scripting.html). This technique was also discussed for persistence purposes by noncetonic in [this post]({% post_url 2019-3-6-macOS-persistence-via-iterm %}). Internally, this is done programmatically using [NSUserAppleScriptTask](https://developer.apple.com/documentation/foundation/nsuserapplescripttask?language=objc). 

# The Vulnerability
This allows us to have iTerm2 send AppleEvents to itself (or other applications) to do things on our behalf. iTerm2 may not be approved to send AppleEvents to other applications, but an app can always send events to itself. So, anything iTerm2 is approved via TCC to do, we are also approved to do.

We can craft an `AutoLaunch.scpt` that asks iTerm2 to exfiltrate the contents of protected directories like `~/Documents`, `~/Downloads`, and `~/Desktop` when the user launches iTerm2 to a directory that we have full read access to. 

# Simple Proof of Concept
Write the following contents to `~/Library/Application Support/iTerm2/Scripts/AutoLaunch.scpt`.
```applescript
-- ~/Library/Application Support/iTerm2/Scripts/AutoLaunch.scpt
tell application "iTerm"
    do shell script "cp -R ~/Documents/ /tmp/"
end tell
```

![Bypass in Action](/images/iterm_tcc.png)


# Caveats
An important note is that `AutoLaunch.scpt` is only executed when iTerm2 first starts up. If it's already running on the target, you may have to be noisy and kill or hang the process, prompting the user to re-open it. 
