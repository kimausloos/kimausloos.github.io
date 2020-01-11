---
layout: post
title:  "Experimenting with Silex"
date:   2014-03-02 14:04:54
---

[Silex](http://silex.sensiolabs.org/) is quite nice to create a small proof of concept, but it's not perfect. One of the small things I dislike is having to manually create a Response object and render the view. Luckily fixing this was quite easy.


You can add an eventlistener and catch the KernelEvents::VIEW event that is dispatched by the HttpKernel module. The event is only fired when a non Response object is returned by the controller. We can fetch the controller from the Request object, generate a path to a view, and render it. Now we can just return an array() in the controller method and still get a nice html view, just like in the full symfony framework.

{% gist 9298197 %}
