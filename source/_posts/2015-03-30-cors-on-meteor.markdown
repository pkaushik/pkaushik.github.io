---
layout: post
title: "CORS on Meteor"
date: 2015-03-30 21:46:59 -0500
comments: true
categories: [meteor, webapps]
---
Meteor's [webapp](http://docs.meteor.com/#/full/webapp) package exposes the underlying [connect](https://github.com/senchalabs/connect) API through `WebApp.connectHandlers` which can be used (among other things) to customize HTTP headers and enable [CORS](http://enable-cors.org/) in a Meteor application. 

``` js
// Listen to incoming HTTP requests, can only be used on the server
WebApp.connectHandlers.use(function(req, res, next) {
  res.setHeader("Access-Control-Allow-Origin", "*");
  return next();
});
```

Use the optional `path` argument to call the handler only for paths that match a specified string. 

```
// Listen to incoming HTTP requests, can only be used on the server
WebApp.connectHandlers.use("/public", function(req, res, next) {
  res.setHeader("Access-Control-Allow-Origin", "*");
  return next();
});
```

