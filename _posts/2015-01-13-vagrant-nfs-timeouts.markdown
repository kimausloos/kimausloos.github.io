---
layout: post
title:  "Vagrant nfs share timeouts on OSX"
date:   2015-01-13 19:05:00
---

I was setting up a vagrantfile and for some reason nfs would not work at all. I was always getting the same error (after a good, long wait of course):

```
Mounting NFS shared folders...
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!
```

I checked the `/etc/exports` file and sure enough, it contained everything to mount the nfs share so this wasn't a permission problem. The nfs service was also running so that couldn't be the issue either.

Long story short: if you have this issue check your mac's firewall settings. You can find it under `System Preferences > Security & Privacy`. If you enabled `Block all incoming connections` under the `Firewall Options` button, nfs will fail to connect.