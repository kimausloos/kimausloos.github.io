---
layout: post
title:  "php-fpm ondemand with systemd"
date:   2014-04-12 14:04:54
---

After reading [Mattias Geniar's blogpost on how to properly run php-fpm](http://mattiasgeniar.be/2014/04/09/a-better-way-to-run-php-fpm/) I saw Stéphan mentioning [it would be cool to be able to spin up fpm master pools on demand](http://mattiasgeniar.be/2014/04/09/a-better-way-to-run-php-fpm/comment-page-1/#comment-9544). This would absolutely minimize all overhead and optimize for low traffic hosting. I knew it was possible to do pretty cool stuff with systemd and sockets but never tried it. Turns out you can do pretty cool stuff allright, and starting up php-fpm pools on demand isn't that hard.

First of all I made a Vagrantbox with the latest Arch linux (2014.04.01) and booted it. Next I installed php, php-fpm and nginx. I didn't try this with apache but I don't expect any issues with that setup either.

Next I edited nginx.conf and added a php handler:

```
location ~* \.php$ {
    fastcgi_index   index.php;
    fastcgi_pass    127.0.0.1:8888;
    include         fastcgi_params;
    fastcgi_param   SCRIPT_FILENAME    /srv/http/$fastcgi_script_name;
    fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
}
```

As you can see the php files are in /srv/http/ and I use a fastcgi_pass to a port. Next I created a very basic php-fpm configuration for a pool called test-pool:

```
[root@vagrant-arch ~]# cat /etc/php/fpm.d/test-pool.conf
[global]
pid = /run/test-pool.pid
error_log = syslog
daemonize = no

[www]
listen = /var/run/test-pool.socket
user = nginx
group = nginx
pm = ondemand
pm.max_children = 10
slowlog = syslog
[root@vagrant-arch ~]#
```

Now the systemd magic begins, according to the [manual](http://www.freedesktop.org/software/systemd/man/systemd.socket.html): creating a socket with a service will allow you to start the service on demand:

```For each socket file, a matching service file must exist, describing the service to start on incoming traffic on the socket```

We create the config file for the socket:

```
[root@vagrant-arch ~]# cat /etc/systemd/system/test-pool.socket
[Socket]
ListenStream=8888

[Install]
WantedBy=sockets.target
[root@vagrant-arch ~]#
```

And the service:

```
[root@vagrant-arch ~]# cat /etc/systemd/system/test-pool.service
[Service]
ExecStart=/usr/bin/php-fpm --fpm-config=/etc/php/fpm.d/test-pool.conf
User=nginx
Group=nginx
Environment="FPM_SOCKETS=/var/run/test-pool.socket=3"
[root@vagrant-arch ~]#
```

Our systemd configuration is in place and we should enable and start our socket, plus enable and start nginx:

```
[root@vagrant-arch ~]# systemctl enable test-pool.socket
ln -s '/etc/systemd/system/test-pool.socket' '/etc/systemd/system/sockets.target.wants/test-pool.socket'
[root@vagrant-arch ~]# systemctl start test-pool.socket
[root@vagrant-arch ~]# systemctl enable nginx
ln -s '/usr/lib/systemd/system/nginx.service' '/etc/systemd/system/multi-user.target.wants/nginx.service'
[root@vagrant-arch ~]# systemctl start nginx
[root@vagrant-arch ~]#
```

The final step is to add a php file to the document root:

```
[root@vagrant-arch ~]# cat /srv/http/info.php
<?php
phpinfo();
[root@vagrant-arch ~]#
```

Now just request info.php:

![Our info.php](/assets/phpfpm-systemd.png)


As you can see the test-pool service was started and has a single child process:

```
[root@vagrant-arch ~]# systemctl status test-pool.service
* test-pool.service
   Loaded: loaded (/etc/systemd/system/test-pool.service; static)
   Active: active (running) since Sat 2014-04-12 15:32:24 UTC; 1min 17s ago
 Main PID: 3446 (php-fpm)
   Status: "Processes active: 0, idle: 0, Requests: 0, slow: 0, Traffic: 0req/sec"
   CGroup: /system.slice/test-pool.service
           |-3446 php-fpm: master process (/etc/php/fpm.d/test-pool.conf)
           `-3451 php-fpm: pool www

Apr 12 15:32:24 systemd[1]: Started test-pool.service.
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: using inherited socket fd=3, "/var/run/test-pool.socket"
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: fpm is running, pid 3446
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: ready to handle connections
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: systemd monitor interval set to 10000ms
[root@vagrant-arch ~]#
```

Killing the php-fpm process has to happen manually as php-fpm (of course) has no option of having it killed when not in use. We can do that with a small shell script we can put in a cron job:

```
[root@vagrant-arch ~]# cat pool-kill.sh
#!/bin/bash

CHILDREN=$(ps -o pid --no-headers --ppid `cat /run/test-pool.pid` | wc -l)

if [ $CHILDREN == "0" ]
then
    kill `cat /run/test-pool.pid`
fi
[root@vagrant-arch ~]#
```

After running it, you can see it kills our php-fpm master process when it has no child processes:

```
[root@vagrant-arch ~]# ./pool-kill.sh
[root@vagrant-arch ~]# systemctl status test-pool.service
* test-pool.service
   Loaded: loaded (/etc/systemd/system/test-pool.service; static)
   Active: inactive (dead) since Sat 2014-04-12 15:38:09 UTC; 2s ago
  Process: 3446 ExecStart=/usr/bin/php-fpm --fpm-config=/etc/php/fpm.d/test-pool.conf (code=exited, status=0/SUCCESS)
 Main PID: 3446 (code=exited, status=0/SUCCESS)
   Status: "Processes active: 0, idle: 0, Requests: 0, slow: 0, Traffic: 0req/sec"

Apr 12 15:32:24 systemd[1]: Started test-pool.service.
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: using inherited socket fd=3, "/var/run/test-pool.socket"
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: fpm is running, pid 3446
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: ready to handle connections
Apr 12 15:32:24 php-fpm[3446]: [12-Apr-2014 15:32:24] NOTICE: systemd monitor interval set to 10000ms
Apr 12 15:38:09 php-fpm[3446]: [12-Apr-2014 15:38:09] NOTICE: Terminating ...
Apr 12 15:38:09 php-fpm[3446]: [12-Apr-2014 15:38:09] NOTICE: exiting, bye-bye!
[root@vagrant-arch ~]#
```

But as soon as we request the phpinfo page it starts up again:

```
[root@vagrant-arch ~]# systemctl status test-pool.service
* test-pool.service
   Loaded: loaded (/etc/systemd/system/test-pool.service; static)
   Active: active (running) since Sat 2014-04-12 15:39:01 UTC; 2s ago
 Main PID: 3464 (php-fpm)
   Status: "Processes active: 0, idle: 0, Requests: 0, slow: 0, Traffic: 0req/sec"
   CGroup: /system.slice/test-pool.service
           |-3464 php-fpm: master process (/etc/php/fpm.d/test-pool.conf)
           `-3465 php-fpm: pool www

Apr 12 15:39:01 systemd[1]: Starting test-pool.service...
Apr 12 15:39:01 systemd[1]: Started test-pool.service.
Apr 12 15:39:01 php-fpm[3464]: [12-Apr-2014 15:39:01] NOTICE: using inherited socket fd=3, "/var/run/test-pool.socket"
Apr 12 15:39:01 php-fpm[3464]: [12-Apr-2014 15:39:01] NOTICE: fpm is running, pid 3464
Apr 12 15:39:01 php-fpm[3464]: [12-Apr-2014 15:39:01] NOTICE: ready to handle connections
Apr 12 15:39:01 php-fpm[3464]: [12-Apr-2014 15:39:01] NOTICE: systemd monitor interval set to 10000ms
[root@vagrant-arch ~]#
```

There we go, the php-fpm pool is only started when needed and won't use any of your resources.

Feel free to send me a message if you have questions: [@kamme](https://twitter.com/kamme)
