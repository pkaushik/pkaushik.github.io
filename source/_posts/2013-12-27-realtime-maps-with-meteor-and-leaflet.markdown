---
layout: post
title: "Realtime Maps with Meteor and Leaflet"
date: 2013-12-27 20:51:10 -0600
comments: true
categories: [meteor, webapps, leaflet]
---
The [Meteor Parties example](https://www.meteor.com/examples/parties) is a compelling demonstration of Meteor's capabilities. Out of the box, it uses a static [jpeg image of SOMA](https://github.com/meteor/meteor/blob/devel/examples/parties/public/soma.png) as a faux base map layer and [D3](http://d3js.org/) to draw circles representing parties with labels showing the number of RSVPs. A neat D3 animation effect indicates which party is currently selected, but since the "map" is just a static image, parties are located at fixed points within the 500x500 pixel image, and you cannot pan or zoom the map.

I decided to refactor the example to use [Leaflet](http://leafletjs.com/) to make a truly interactive realtime map that looked and felt as close to the original parties demo as possible. 

TLDR; Here is the [end result](http://chicago-parties.meteor.com) with code for the complete application in this  [github repo](https://github.com/pkaushik/parties). You need a github account to sign in to the app and post or be invited to parties. The rest of this post describes how I got it all to work, and assumes you have a working understanding of the original Parties example. Please comment below if you want a more detailed explanation.

<h3>Base Map</h3>

First off, Stamen's [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) are an obvious replacement for the black & white SOMA image. I set ```opacity``` to ```.5``` to improve the visibility of the markers and initialized the map with ```doubleClickZoom``` and ```touchZoom``` set to ```false``` since those actions are used to create a new party. All of this Leaflet initialization goes into the ```rendered()``` callback of the map template. 

{% codeblock lang:js %}
map = L.map($('#map_canvas'), {
  doubleClickZoom: false,
  touchZoom: false
}).setView(new L.LatLng(41.8781136, -87.66677956445312), 13);

L.tileLayer('http://{s}.tile.stamen.com/toner/{z}/{x}/{y}.png', {opacity: .5}).addTo(map);
{% endcodeblock %}

In addition, the following boilerplate adjusts the map's size when the window is resized.

{% codeblock lang:js %}
$(window).resize(function () {
  var h = $(window).height(), offsetTop = 90; // Calculate the top offset
  $mc = $('#map_canvas');
  $mc.css('height', (h - offsetTop));
}).resize();
{% endcodeblock %}

Finally, the map template event handler from the example (which triggers when the image is clicked) is replaced with Leaflet's ```dblclick``` event handler (which conveniently returns a Leaflet ```LatLng``` as argument to the callback). I save the ```LatLng``` as a reactive ```Session``` variable and set a flag to render the ```createDialog``` template.

{% codeblock lang:js %}
map.on("dblclick", function(e) {
  if (! Meteor.userId()) // must be logged in to create parties
    return;
  
  Session.set("createCoords", e.latlng);
  Session.set("showCreateDialog", true);
});
{% endcodeblock %}

<h3>Creating a Party</h3>

The code associated with the ```createDialog``` template is virtually the same as in the Parties example; except that my new ```createParty``` takes ```LatLng``` as an argument instead of the x and y coordinates in the example.

{% codeblock lang:js %}
Meteor.call('createParty', {
  title: title,
  description: description,
  latlng: latlng,
  public: public
}, function (error, partyId) {
  if (! error) {
    Session.set("selected", partyId);
    if (! public && Meteor.users.find().count() > 1)
      Session.set("showInviteDialog", true);
  }
});
{% endcodeblock %}

If ```createParty``` executes without error, I save the newly created Party's ```_id``` to a reactive ```Session``` variable and set a flag to render the ```inviteDialog``` template.

<h3>Observing the Parties Collection</h3>
The original example's D3 code contained in a ```Meteor.autorun``` which runs when a Party is created or updated with the following code in the map template's ```created()``` callback.

{% codeblock lang:js %}
Template.map.created = function() {
  Parties.find({}).observe({
    added: function(party) {
      var marker = new L.Marker(party.latlng, {
        _id: party._id,
        icon: createIcon(party)
      }).on('click', function(e) {
        Session.set("selected", e.target.options._id);
      });      
      addMarker(marker);
    },
    changed: function(party) {
      var marker = markers[party._id];
      if (marker) marker.setIcon(createIcon(party));
    },
    removed: function(party) {
      removeMarker(party._id);
    }
  });
}
{% endcodeblock %}

The ```addMarker``` and ```removeMarker``` methods update a hash of Markers keyed by the Party's ```_id```. This is useful since we need to efficiently access Markers by ```Party._id``` to update RSVP count in the Marker's laber or delete the Marker if the Party is deleted.

The ```createIcon``` helper function takes a Party document as argument, and returns a pure CSS circular ```DivIcon``` created using ```border-radius``` to generte a circle. 

{% codeblock lang:js %}
var createIcon = function(party) {
  var className = 'leaflet-div-icon ';
  className += party.public ? 'public' : 'private';
  return L.divIcon({
    iconSize: [30, 30],
    html: '<b>' + attending(party) + '</b>',
    className: className  
  });
}
{% endcodeblock %}

The ```attending``` helper function is under the ```/collections``` directory since it's used on both the client and server side of the application.

{% codeblock lang:js %}
attending = function (party) {
  return (_.groupBy(party.rsvps, 'rsvp').yes || []).length;
};
{% endcodeblock %}

<h5>Next: Drawing circle markers and animating which one's selected...</h5> 

