---
layout: post
title: "Realtime Maps with Meteor and Leaflet - Part Two"
date: 2013-12-28 17:53:06 -0600
comments: true
categories: [meteor, webapps, leaflet]
---
{% imgcap left /images/custom/updated-parties.png this is a Leaflet map with DivIcon markers %} 
<h3>Recap</h3>
In the [last post]({{root_dir}}/blog/2013/12/27/realtime-maps-with-meteor-and-leaflet/), I initialized a Leaflet map to work with [Stamen Design](http://stamen.com/)'s [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) and [Bootstrap's responsive layout](http://getbootstrap.com/2.3.2/scaffolding.html#responsive). I then set up a double-click event handler to gather additional details about the new party, and hooked up the dialog's save button to pass those details to a `Meteor.methods()` call to save the party into a server-side mongo collection. Finally, I hooked up a `cursor.observe()` `added()` callback to the client-side minimongo collection and set up the callback to automatically add a circular `DivIcon` marker at the specified coordinates. 
<!--more-->
<h3>Updating Party Details in the Database</h3>
A party document saved to mongo looks something like this:
``` js
{
  _id: "22dQwpajD64LCv4QW",
  title: "1871",
  description: "Party like it's 1871!",
  latlng: {
    lat: 41.88298161317542,
    lng:  -87.63811111450194
  },
  public: false,
  owner: "52xdsNjprquesL2tQ",
  invited: ["52xdsNjprquesL2tQ", "ci7bzkJCpH9R7HCZK", "5qhRdKFcsmPnxZKBr"]
  rsvps: [
    {
      rsvp: "yes",
      user: "52xdsNjprquesL2tQ"
    },
    {
      rsvp: "maybe",
      user: "ci7bzkJCpH9R7HCZK"
    }
  ]
}
```
Each party contains an array of RSVP objects, which must be updated when any user updates their RSVP to the party. In addition, private parties contain a set of invited userIds. So `rsvps` and `invited` are the two mutable party attributes in our example. The owner, title, description, coordinates or public/private setting cannot be changed, but a party's owner can delete the party if no user is RSVPd as Yes (Maybes don't count).

The code to update and delete parties in the server-side mongo collection is virtually unchanged from the original. The `invite()` and `rsvp()` template event handlers are hooked to `Meteor.methods()` calls that perform the necessary checks before updating the mongo collection on the server. As usual, behind the scenes, Meteor updates the client-side minimongo collection as soon as the server collection is updated.

<h3>Updating and Removing Map Markers in Realtime</h3>
I hooked up the `cursor.observe()` `changed()` callback to update the party's icon, and `removed()``` callback to delete the marker from the map and from the local `markers` hash.

``` js
var map, markers = {};

Template.map.created = function() {
  Parties.find({}).observe({
    added: function(party) {/* see previous post */},
    changed: function(party) {
      var marker = markers[party._id];
      if (marker) marker.setIcon(createIcon(party));
    },
    removed: function(party) {
      var marker = markers[party._id];
      if (map.hasLayer(marker)) {
        map.removeLayer(marker);
        delete markers[party._id];
      }
    }
  });
}
```
<h3>Using a Halo Marker to Indicate Which Party Is Selected</h3>
{% imgcap right /images/custom/selected-party.png a selected party %} 

Up to this point, there's been no visual indication _on the map_ as to which party is currently selected. Just like in the original Parties example, I solved this by creating a 50px x 50px transparent grey circular marker and making it concentric with the currently selected party's marker such that it formed a 20px halo around the selected party. The halo marker is purely a UI artefact that does not need to be saved on the server.

``` js
L.divIcon({
  iconSize: [50, 50], // set to 50px x 50px
  className: 'leaflet-animated-div-icon'
}
```
``` js
.leaflet-animated-div-icon {
  border-radius: 50%;
  border: none;
  opacity: .2;
  background: black;
}
```

<h3>Animating the Halo Marker</h3>
For a final flourish, I used the `AnimatedMarker` Leaflet [plugin](https://github.com/openplans/Leaflet.AnimatedMarker) from [OpenPlans](http://openplans.org/) to animate the halo's movement on the map when a user selects different parties rather than simply making it reappear at a different location. `AnimatedMarker` takes a Leaflet `polyline` object as the first argument to its initialize function, and draws a marker at the beginning of the polyline, which it then animates along the `polyline` at a speed (in meters/ms) that's configurable via a second argument. 

I needed to make a minor tweak to the plugin's source code to support my needs: `AnimatedMarker` does not allow setting the animation `polyline` after the marker is initialized. In other words, it requires the animation path to be known before creating the marker. I wanted to create the marker around the currently selected party without knowledge of it's future animation path, and to set the animation path dynamically as soon as a user selected a different marker -- the path would be a segment from the current location to the center of the selected marker. To accomplish this, all I needed to do was reset the animation index in the marker's `setLine` method. This modification is available at [my fork](https://github.com/pkaushik/Leaflet.AnimatedMarker) on github.

And ta-da! This is the end result: http://chicago-parties.meteor.com with [source code for the complete application](https://github.com/pkaushik/parties). You need to log in with a github account to create or RSVP to parties.
