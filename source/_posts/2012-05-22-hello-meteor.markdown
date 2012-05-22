---
layout: post
title: "Hello Meteor!"
date: 2012-05-22 15:22
comments: true
categories: 
---
These are slides from my [JS.Chi() talk](http://www.meetup.com/js-chi/events/59833642) on Meteor. I will follow up with posts expanding on all of the slides in this presentation. This post, however, is for the benefit of all those who are feeling like they might be missing a step with frameworks like Backbone, Ember, (and now Meteor), et. al. Read on for recommendations on what to read up on to understand more about the problems they're solving.
 
<script async class="speakerdeck-embed" data-id="4fbbc61f15a68f001f027e5a" data-ratio="1.2945638432364097" src="//speakerdeck.com/assets/embed.js"></script>

<br/>
<hr/>
<br/>

<h4>First a brief note about MVC</h4> 

The Model-View-Controller application architecture (MVC) is used in most modern applications to separate an application’s data from its business rules and user interface. Briefly, Models represent an application’s data and functions to access it. Views represent information presented to the user. Controllers represent intermediary resources required to generate Views. The MVC pattern isn't new (it can be traced back to Smalltalk), but it was popularized, along with the REST style of client-server API design by server-side web frameworks such as Rails and Django.

<h4>So what's all the fuss about JavaScript MVC frameworks (Backbone, Ember, et. al.)?</h4>

If you think of the screens - or pages - in your application as different "states" in your application, server-side frameworks require a round trip back to the server when the user goes from one application state to another. This is fine for content-heavy apps or websites. Or if you're accessing the pages via an ethernet cable or 802.11. But when you move to high latency connections and along the continuum -- and I believe it really is a continuum -- from a web site to a web app, you can deliver a much better user experience if you avoid the server round trip for each state change, and instead switch from state to state on the client, within a single browser page load cycle.

This saves you all of the overhead of destroying and recreating the DOM tree, flushing browser context (JavaScript, CSS, etc.) on each new page load. You also are transmitting less data on the wire (or ether) since HTML is generated on the client, and only JSON passed over the network. Both of these factors (page refresh time and network traffic optimization) are significant when you consider the fact that more and more internet usage is taking place via mobiles and tablets.

You would still need a server to authenticate users and perform other privileged operations, and to store the "official" version of the data that different clients synchronize with, but instead of sending dynamically created HTML to the client (a la aforementioned server-side frameworks) the server would send as much data as needed (typically as JSON) to the client, and the client would create all of the different application states dynamically. The problem with this approach -- and I've been writing apps this way since 2008, without the benefit of any client side frameworks -- is that they quickly turn into a mass of spaghetti code.

This is where the likes of Backbone, Ember, etc. come in. They are a re-imagination of the MVC application architecture for JavaScript application development in the client, where all (or nearly all) application states are handled in the client, in JavaScript. They provide a way to structure your JavaScript applications in a more organized MVC style.

[Addy Osmani's articles](http://addyosmani.com/largescalejavascript/), and especially his [Backbone Fundamentals book](http://addyosmani.github.com/backbone-fundamentals/) are a great place to start reading more about this. Although it's focused on one specific framework (Backbone), it has a good overview of MVC and how the pattern has moved from server side frameworks (like Rails, Django) to the client side (Backbone, et. al.).

I also wholeheartedly recommend his [recent talks on providing structure to JavaScript apps](http://addyosmani.com/scalable-javascript-videos/).

And this is a useful [comparison of various front-end frameworks](http://codebrief.com/2012/01/the-top-10-javascript-mvc-frameworks-reviewed/), although the conclusions should be taken with a grain of salt (it's one persons's perspective, and things are not quite as quantifiable as the article suggests):


So, all of these are client side frameworks, which solve the problems of a) separating client side code into MVC, and b) optionally, allowing views to "observe" client side models, so that views get updated automatically as the data in the models is updated or as new data arrives from the server.

This gets you pretty far, but all of these applications have something else in common which the previously mentioned MVC frameworks don't address: They need to send data back to the server and check in if data has changed on the server. In other words, they need to synchronize client side models with the server side models.

<h4>Enter Meteor.</h4> 

[Meteor](http://meteor.com) is a full stack framework for end-to-end JavaScript applications. The beauty of coding it all in JavaScript is that you can easily implement the observer pattern (views watching for changes to the models) all the way to the server side. Meteor
apps can subscribe to models (or subsets of models) on the server, and meteor takes care of all of the plumbing. This really is the bleeding edge of JS today.

For more on the problems Meteor solves, read the [meteor docs](http://docs.meteor.com/) from the intro section through to as many of the detailed concepts you're interested in. Or come to [my talk at JS.Chi()](http://www.meetup.com/js-chi/events/59833642) and we will work through it together.

Have fun!