---
author: noncetonic
layout: post
title: Chrome Arbitrary Javascript Injection Via AppleScript
tags: [MacOS, Post-Exploitation]
---

## Foreword ##
I've had a PoC for this technique for a little while as part of research for a future talk on the subject of post-exploitation in MacOS, but as an article has surfaced [detailing the adware OSX.Pirrit][4]'s use of this technique, I thought it appropriate to disclose this writeup. It should be noted that while this particular PoC targets only Google Chrome, the attack works against Safari as well. Mozilla Firefox is exempt from this specific attack as they do not expose a similar scripting definition to AppleScript.

## Browser hijacks are hard, let's go shopping ##
Wouldn't it be nice to inject arbitrary Javascript into a user's browser regardless of XSS detection, website, or hoping a user is foolish enough to visit a BeEF link? 

Sniffing credentials right out of form submissions, disregarding 2FA by leveraging a victim's already active tab, redressing/overlay attacks and much more are possible with this technique, and all done with no specialized tools.

Join me as I rush to release this post before everyone writes detection around this surprisingly well documented feature of Chrome/Safari on MacOS and create a mostly harmless inject which sneakily scrapes a Gmail tab for Email, Sender name, and Subject informatino leveraging Applescript as an In-Memory Javascript Injector

## Has anyone really ever been as far as to do want inject javascript? ##

### The PoC ###
You probably don't care about the how and just want a pre-packed PoC ready to be used for rickrolling your friends so here it is:

Full Chrome Gmail Javascript Injection AppleScript
<script src="https://gist.github.com/n0ncetonic/aa4a523d01c822953d0d0cb50d5ac7c0.js?file=Chrome_Gmail_Javascript_Injection.applescript"></script>

Non-Minified Javascript payload
<script src="https://gist.github.com/n0ncetonic/aa4a523d01c822953d0d0cb50d5ac7c0.js?file=chrome_arbitrary_javascript_injection_PoC.js"></script>

Minimized OSAScript oneliner
<script src="https://gist.github.com/n0ncetonic/aa4a523d01c822953d0d0cb50d5ac7c0.js?file=minified_osascript_payload.sh"></script>

Minified Javascript payload (keeps size of Applescript payload/oneliner small)
<script src="https://gist.github.com/n0ncetonic/aa4a523d01c822953d0d0cb50d5ac7c0.js?file=minified_chrome_arbitrary_javascript_injection_PoC.js"></script>

### How the whut? ###
Put simply, AppleScript is boss and very few built-in scripting languages on other Operating Systems come close to the amount of power that given to end-users and developers. Intended primarily to be used for automation, AppleScript has the potential to open the door to a lot of fun for system administrators and hackers alike.

Applications with dedicated integrations with AppleScript contain an `.sdef` file which outlines the _S_cripting _Def_initions available to AppleScript. Poking around a bit on my system I noticed that Chrome and Safari had .sdef files. 

[Here's a link to Chrome's for those who want to follow along][1]
For a little more fun [Here is an example provided by Google themselves on executing Javascript in the browser][2]

Writing a little bit of AppleScript to take advantage of this feature and leveraging a [Javascript minimizer][3] to keep the size of the payload small results in a really fun little browser hijack technique.

### What's it do? ###
While a primer on AppleScript is outside the scope of this post, a quick breakdown of the code is provided as comments in the provided code to aid the reader.

RTFM


## Closing ##
While I'm sad that I wasn't the first one to weaponize this technique publicly I hope that this public release of a fairly harmless PoC will spark discussion around writing more creative payloads and help excite malware authors and security researchers to take more time to look at MacOS and develop more post-exploitation techniques.

I leave implementation of this PoC in Safari as an exercise for the reader.

RIP Trevor; another fun bug squashed in public.

[1]:https://chromium.googlesource.com/chromium/src.git/+/master/chrome/browser/ui/cocoa/applescript/scripting.sdef
[2]:https://chromium.googlesource.com/chromium/src.git/+/master/chrome/browser/ui/cocoa/applescript/examples/execute_javascript.applescript
[3]:http://jsbeautifier.org/
[4]:https://www.cybereason.com/blog/targetingedge-mac-os-x-pirrit-malware-adware-still-active
