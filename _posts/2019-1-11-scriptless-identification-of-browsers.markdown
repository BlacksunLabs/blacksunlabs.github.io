---
author: noncetonic
layout: post
title: Scriptless Identification of Browsers
---

## Foreword ##
This is a fairly short post illustrating a technique for identifying a browser based on request headers. This technique is not reliant on any scripting support and sucessfully identifies browsers regardless of User-Agent spoofing or other such obfuscation techniques.

Provided alongside this post is a PoC tool that implements the checks mentioned in this post.

## Background ##
Request headers sent by browsers are fairly standard but due to implementation details in individual browser codebases provide metadata unique to the browser which can be used to positively identify the browser.

As request headers can be read server-side without the need for any client-side scripting capabilities this technique is useful for identifying a browser which has extentions/plugins installed aimed at disrupting fingerprinting.

## Where them headers at? ##
Here are some examples of different request headers sent by Chrome, Firefox, and Safari. 

### Chrome ###
```
GET / HTTP/1.1
Host: localhost:8081
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
```

### Firefox ###
```
GET / HTTP/1.1
Host: localhost:8081
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:62.0) Gecko/20100101 Firefox/62.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```

### Safari ###
```
GET / HTTP/1.1
Host: localhost:10101
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0.3 Safari/605.1.15',
Accept-Language: en-us
Accept-Encoding: gzip, deflate
```

### GG noncetonic, so what? ###
Astute readers may notice the following:

- After the `GET` request and `Host` header all browsers use the same handful of headers but with unique order.
- Some headers contain details in one browser that are not in other browsers.
- Some browsers use different case in headers which other browsers do not.

When these points are combined, the following fingerprints are derived.

### Identifying Chrome ###
- Unique order of request headers
- `Accept-Encoding` lists `br` as an encoding which allows support for brotli encoding https://en.wikipedia.org/wiki/Brotli
- `Accept` adds two mime-types not typically seen explictly stated with other browsers (despite other browsers having support for these mime-types) : `image/webp` and `image/apng`.
- `Accept-Language` has a modified `q` value (currently unsure of the significance of the q variable) of 0.9 contrasting the 0.5 of Firefox and the complete omission of the `q` variable in Safari

### Identifying Safari ###
- Unique order of request headers
- `Accept-Language` lists only `en-us` and omits the `,en;q=0.X` format of Chrome and Firefox. Additionally, both Chrome and Firefox use the spelling `en-US` but Safari uses `en-us` instead.

### Identifying Firefox ###
- Unique order of request headers

## Damn, Daniel! ##
Taking the above rules and creating fingerprints based on them is extremely easy. Here are arrays for each of these three browsers containing the order of request headers unique to that browser.

```
chrome = ["host","connection","upgrade-insecure-requests","user-agent","accept","accept-encoding","accept-language"]
firefox= ["host","user-agent","accept","accept-language","accept-encoding","connection","upgrade-insecure-requests"]
safari= ["host","connection","upgrade-insecure-requests","accept","user-agent","accept-language","accept-encoding"]
```

One thing to note is the inclusion of certain headers such as `DNT` and `Pragma` which are not seen unless settings are enabled in the browser or a browser is explictly forcing a non-cached request to a page. For these reasons certain headers have been ignored.

## My favorite rapper is 2-PoC ##
(Ok, so there aren't really 2 PoCs and I don't really like 2-Pac but I needed a pun for this second title.)

I've shared a PoC node.js server that outputs the detected browser using both request header order as well as unique header details. You can check it out at https://github.com/n0ncetonic/browseRekt .

## Closing ##
As a simple PoC there are definitely browsers missing and more research that could be done; that is an exercise left to the reader. Something worth noting is that as many browsers base their code on the open-source Chromium project (Opera, Vidalia, etc.) this technique will inaccurately assume these browsers to be Chrome. This is a limitation of this technique and other identification methods must be leveraged in order to accurately determine whether the browser is in fact Chrome or a branched code base.
