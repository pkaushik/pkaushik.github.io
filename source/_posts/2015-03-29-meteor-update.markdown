---
layout: post
title: "Meteor Update"
date: 2015-03-29 22:27:33 -0500
comments: true
categories: [meteor, google]
---
{% imgcap left /images/custom/googoo.jpg this happened %} 

This picture sums up why I haven't been blogging for a while -- I joined Google as a software engineer on the Search team (more on this later) and I made a baby! 

In other news, my realtime maps example now works with Meteor 1.0.5. All that I needed to do was:

1. `meteor update`
2. Move template methods into a `Template.helpers()` call - here is an [example diff](https://github.com/pkaushik/parties/commit/42295a2896237d953a5d5ff2a846ab474103aec2)
3. [Remove](https://github.com/meteor/meteor/wiki/Using-Blaze#no-more-constant-isolate-or-preserve) the `#constant` guard around the map div

[Code here](https://github.com/pkaushik/parties) and [demo here](http://chicago-parties.meteor.com/).
