---
layout: post
title:  "Weird 5 second Silex performance issues?"
date:   2014-03-11 22:36:37
---

While I was experimenting with Silex I noticed requests tended to take a pretty long time to finish. I didn't really notice it at first, but when I was rendering a very simple twig template it struck me: rendering this shouldn't take 5 seconds.

When looking at the code I didn't notice anything weird, so I began doing some tests and saw all requests took the same time: around 5 seconds. I figured I'd have a look at my headers and see if it was the browser or my app and fired up curl. That gave me an interesting error:

```
curl: (18) transfer closed with outstanding read data remaining
```
This raised more questions and google couldn't really help me so I started timing steps and debugging my Silex app but even returning a string gave the same 5 seconds wait time.

I started thinking about what steps are taken when you fire a http request to your server and then it hit me. The .htaccess file is always parsed on every request and that proved to be the culprit. Replacing my one-line ```FallbackResource /app.php``` with a more standard ```RewriteRule``` version fixed the issue.

Ah well, I hope this saves some other poor webdeveloper some time.
