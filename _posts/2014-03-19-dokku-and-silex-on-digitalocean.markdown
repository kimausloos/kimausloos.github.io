---
layout: post
title:  "Dokku and Silex (on DO)"
date:   2014-03-19 14:04:54
---

I installed [dokku](https://github.com/progrium/dokku) on a [DigitalOcean]( https://www.digitalocean.com/?refcode=98ac45b0412a ) droplet a couple of months ago to play around with some nodejs apps but removed the droplet after a while when I was done. Today I wanted to try something out in node and installed a new droplet with dokku. After fiddling with the node app I thought it would be cool to see if [Silex](http://silex.sensiolabs.org/) would also run on it. Turns out it does, and it's quite easy to get it working too.

When you're logged in on your DigitalOcean account just click 'Create' to make a new droplet. Fill in a hostname and choose your region. Next instead of choosing a linux distro, you choose an application from the second tab. We of course want dokku. If you already have a ssh key linked to your DO account you're pretty much done, just select it and click "Create Droplet".

![Create droplet](/assets/dokku-on-do.png)

After a minute when your droplet is created, just open your browser and go to the droplet's ip. You'll get a screen like this:

![Configure dokku](/assets/dokku-configuration.png)

When you selected an existing ssh key it should be filled in here, otherwise you should fill one in now. Also fill in the domain name you want to use. You can also fill in a subdomain if you just want to try it out, just make sure you add a wildcard dns entry for all sub-subdomains to the droplet ip. You'll also want to check the "use virtualhost naming" option, otherwise you'll end up with apps being deployed to ports on your droplet. The virtualhost naming adds a subdomain based on the app name. Just click "Finish Setup" when you're done.

Now you're ready to handle the Silex part. Create a new folder with these files:

{% gist 9652146 %}

This is just a very basic Silex app, we'll call it 'IP', that returns your public ip address. Very basic, but you get the idea.

When you're done just do a ```git init```, add and commit your files. Next you should add a remote that's pointing to your dokku server: ```git remote add dokku dokku@DOMAIN.COM:ip```. Now you can do ```git push dokku master``` to deploy your master branch to the remote called 'dokku' that we added before. You'll get some output and one of the last lines should be the new url of your application. Visiting it in your browser will give you your public ip address.

Congrats, you just deployed a Silex app on dokku!
