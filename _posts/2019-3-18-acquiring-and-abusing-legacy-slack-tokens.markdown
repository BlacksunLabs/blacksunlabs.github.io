---
author: noncetonic
layout: post
title: Acquiring and Abusing Slack Legacy Tokens on macOS
title: [MacOS, Post-Exploitation]
---

## Foreward 

Right on the tails of my last post that detailed a token theft attack on macOS, today I'm sharing yet another procedure for acquiring and abusing tokens found on a macOS system.

Included with this post are two github projects, one which helps extract the tokens from Slack, and another which leverage the tokens for incognito logging and monitoring of a Slack team.

## Legacy Tokens

So what exactly are "Legacy tokens" ?

> These tokens were associated with legacy custom integrations and early Slack integrations requiring an ambiguous "API token." They were generated using the legacy token generator and are no longer recommended for use. **They take on the full operational scope of the user that created them**. If you're building a tool for your own team, we encourage creating an internal integration with only the scopes it needs to work.
_From: https://api.slack.com/docs/token-types#legacy_

Slack's documentation on these tokens is unfortunately useless, save the mention that they are scoped to the individual user rather than the team as a whole (similar to what we would see with bot users).

What I've managed to piece together is that Legacy Tokens work for a large portion of Slack's current API endpoints but has some hiccups with more modern endpoints such as some of the calls to the Conversations API. The RTM (Real Time Messaging) API in particular appears to be the perfect usecase for Legacy Tokens.

One thing that became quickly obvious was that the Slack desktop client which heaviliy leverages a websocket connection for communication with the Slack servers mimicks a real-world implementation of the RTM API and during the initial login by a user to a team is making use of OAUTH to generate a valid token for the user which will allow the desktop client to provide the user with all of its functionality.

It is during this authentication flow that I believe the Slack desktop client is granted a Legacy Token due to it being Slack's official client; despite the fact that these tokens are not generated for developers and Slack encourages anyone with a Legacy Token to upgrade to more modern tokens. Best of all, it appears that reusing a Legacy Token allows bypassing 2FA/SSO.

Perfect :)

Legacy tokens can be identified by starting with the characters `xoxs-`. This is in contrast to User tokens (`xoxp-`), Bot tokens (`xoxb-`), and Workspace tokens (`xoxa-2` and `xoxr-`).

## Acquiring Legacy Tokens

Slack stores a copy of currently used Legacy Tokens for a user across all the teams the user is actively authenticated to. These tokens can be found in the LevelDB cache stored on disk and located on macOS at `$HOME/Library/Application\ Support/Slack/Local\ Storage/leveldb/`. 

The LevelDB files appear to use an 8-bit ASCII variant (ASCII is generally a 7-bit character encoding) `ISO-8859-1` which can cause issues when attempting to read the files using a terminal. 

To help mitigate the encoding issue and automate the process of dumping tokens I wrote a script in Ruby called [`toke_em`](https://github.com/n0ncetonic/toke_em).

toke_em using the standard version of ruby installed on macOS with no third party gems so it is ready to run on a system of your choice with no additional dependencies. 

## Abusing Legacy Tokens

Originally I had decided to leave implementation of tokens acquired with toke_em into useful tooling as an exercise to users of toke_em. A few nights ago I realized I should probably write a tool of my own to leverage Legacy Tokens and set off on writing my own terminal-based Slack client with the purpose of covertly monitoring conversations. I also opted to skip implementing any features that could potentially send input that would result in alerting the user in any way.

After I got things working and showed some friends I decided I'd share my client with the internet.

### Respite

Read-only Slack RTM API client for spying on teams.

Respite is a terminal based, read-only client for Slack’s RTM API which authenticates using auth tokens to bypass 2FA and SSO.

![](https://user-images.githubusercontent.com/29786827/54484700-dae76400-4828-11e9-9d53-37111a95ebfe.png)

Respite is still under active development and new features will continue to be added near future and serves as a PoC demo for showcasing the full extent of the usefulness of Legacy Tokens extracted with toke_em. Respite can be downloaded from the [Blacksun Labs github](https://github.com/BlacksunLabs/respite)

## Closing

That's all for now. Make sure to star and watch respite on Github to get updates on new features.

Until next time,
n0ncetonic
