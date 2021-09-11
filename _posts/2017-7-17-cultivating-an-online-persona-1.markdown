---
author: noncetonic
layout: post
title: Cultivating An Online Persona Part 1 - Bypassing Gmail's Phone Verification
tags: [OSINT]
---

## Foreword ##
Cultivating an online persona in the modern era of the internet can be difficult without giving up too much of your own personal information due to the rise of spammers leveraging bots to generate bulk accounts. This leaves the anonymity minded individual with very few options for anonymously registering an account with many services short of paying for services such as recyclable SMS numbers, buying bulk accounts from Phone Verified Account (PVA) shops, or buying SIM cards for one-time use. 

Additionally, services are beginning to prompt users to provide photographic proof of their identity making burner SIM cards and other anonymous SMS verification methods only one part of the problem. 
 
The "Cultivating An Online Persona" series will discuss various methods for obtaining access to popular online services while maintaining a user's anonymity as well as provide readers with an insight into methodologies which can be employed against other services.

## Email Addresses Are Important ##
In a world where your email address is as good as your state ID and act as your gateway to registering for other services, the first step towards cultivating an online persona hinges upon your ability to create an email address which has as little ties to your true identity as possible. 

Because of the importance of email addresses, this post will focus on creating a Gmail account while bypassing phone verification.

## Bypassing Gmail's Phone Verification ##

### Why Gmail? ###
Many readers may wonder why one would even bother attempting to register a Gmail account when there are other services such as [ProtonMail](https://protonmail.com/){:target="_blank"} which allow users to register for accounts with no personal information. In the opinion of others, myself included, registering for an email account on services such as this inherently gets your persona grouped into a subset of "non-standard" internet denizens. Additionally, due to the popularity of Gmail the chances of a Gmail hosted email address being banned from use for signup or being used as a trigger for secondary anti-spam/anti-bot verification (SABV) is substantially lower.

### The Theory ###
There have been many free bypasses for Gmail phone verification throughout the years including things such as signing up from third-world countries where mobile phones are less common to leveraging mobile phone emulator software such as BlueStacks. While these techniques worked in the past the success rate has dropped substantially causing many to abandon them as viable. 

>Note: Using mobile phone emulators is an extremely effective way of sneaking by SABV on services primarily accessed via mobile applications such as Instagram.

One technique which has been used successfully is a very simple one in nature is to register for a Gmail account using the youngest age allowable by [Google's adherence][1] to the [Children's Online Privacy Protection Act of 1998][2]. 

According to Google's age requirement policy, in countries outside of South Korea, Spain, and the Netherlands, registrants must be at least 13 years old to create a Google account. As 13 year olds are not expected to have their own mobile phone for phone verification, providing a secondary email for recovery purposes is enough to bypass phone verification. The sweet spot I've found for age range is between 13 and 15 in countries where 13 is the minimum age requirement.

### The Plan ###
In order to create our Gmail account we need to satisfy a few requirements:

* Birth year that puts you between 13 and 15 years old.
* Email account used to satisfy recovery options

These requirements are easily satisfied and will be discussed before the actual creation of the Gmail account.

### Becoming A Teenager ###
A great resource for all your online persona needs is [FakeNameGenerator][3]. Leveraging FakeNameGenerator it's as easy as clicking the provided link below to generate a birthdate that puts you within our defined age range. All the fun of being a teen again without all the angst and weird body changes!

[Generate A Teenage Birthday][4]

![FakeNameGenerator]({{site.url}}/images/caop_fng.png)

### Getting A Recovery Email ###
There are a lot of places providing email addresses with little to no verification required from a user and a Google search will turn up enough results if you can't think of one. For the purposes of this post we will use [Mail.com][5]. [This link will send you straight to their sign-up page][6]. It's recommended to get your recovery email address to match your desired Gmail address as closely as possible. This isn't a requirement but it doesn't hurt in the off chance your registration is checked by an account review process.

>Note: As this is the email address that will be used to regain access to your Gmail in case of lockout it is advised to use a unique, strong password if you have reason to believe your account would be targeted for takeover by a third-party.

![Recovery Email Signup]({{site.url}}/images/caop_recovery-email.png)

### Registering With Gmail ###
And now the section you've been waiting for.

 1. Visit the [Google Signup Page][7]
 2. Fill out the signup form. Personally I just copy paste the information generated by FakeNameGenerator.
 3. Ensure your ***Birthday*** puts you in the appropriate 13-15 year age range.
 4. Leave the ***Mobile phone*** field blank.
 5. Provide your recovery email in the ***Your current email address*** field.
![Gmail Account Signup]({{site.url}}/images/caop_gmail-signup.png)
 6. Click ***Next step***.
 7. Confirm your recovery email by clicking the link provided by Google that will be sent to your provided recovery email address.
 8. Enjoy your new Gmail account.

>Note: While using Tor will almost instantly get you flagged, attempting to create multiple Gmail accounts in this way using the same IP address and without clearing your browsing session will often result in the registration getting flagged by SABV and causing you to get the phone verification prompt.

## Closing ##
Now that you've got your own real-life verified Gmail account the world is your oyster. The next article in this series will cover some of the services you can use your fake persona with to start creating a believable online identity.

[1]:https://support.google.com/accounts/answer/1350409
[2]:https://en.wikipedia.org/wiki/Children%27s_Online_Privacy_Protection_Act
[3]:http://www.fakenamegenerator.com
[4]:http://www.fakenamegenerator.com/advanced.php?t=country&n%5B%5D=us&c%5B%5D=us&gen=50&age-min=13&age-max=15
[5]:https://mail.com
[6]:https://service.mail.com/registration.html?edition=us&lang=en&#.2859002-header-signup2-1
[7]:https://accounts.google.com/SignUp?service=mail&continue=https%3A%2F%2Fmail.google.com%2Fmail%2F&ltmpl=default
