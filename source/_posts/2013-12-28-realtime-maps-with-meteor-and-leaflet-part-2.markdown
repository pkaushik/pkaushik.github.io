---
layout: post
title: "Realtime Maps with Meteor and Leaflet - Part Two"
date: 2013-12-28 17:53:06 -0600
comments: true
categories: [meteor, webapps, leaflet]
---
{% imgcap left /images/custom/updated-parties.png this is a Leaflet map with DivIcon markers %} 
<h3>Recap</h3>
In the [last post]({{root_dir}}/blog/2013/12/27/realtime-maps-with-meteor-and-leaflet/), we initialized a Leaflet map to work with [Stamen Design](http://stamen.com/)'s [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) and [Bootstrap's responsive layout](http://getbootstrap.com/2.3.2/scaffolding.html#responsive). We then set up a double-click event handler to gather additional details about the proposed new party, and we hooked up the dialog's save button to pass those details to a ```Meteor.methods()``` call to save the party into a server-side mongo collection. Finally, we hooked up a ```cursor.observe()``` ```added()``` callback to a query on the client-side minimongo collection and set up the callback to automatically add a circular ```DivIcon``` marker at the specified coordinates. 

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
Each party contains an array of RSVP objects, which must be updated when any user updates their RSVP to the party. In addition, private parties contain a set of invited userIds. So ```rsvps``` and ```invited``` are the two mutable attributes of a party; the owner, title, description, coordinates or public/private setting cannot be updated as per the original example. 

In addition, a party's owner can delete the party if no user is RSVPd as Yes (Maybes don't count).

The code to update and delete parties in the server-side mongo collection is virtually unchanged from the original. The ```invite()``` and ```rsvp()``` template event handlers are hooked to ```Meteor.methods()``` calls that perform the necessary checks before updating the mongo collection on the server. As usual, Meteor, behind the scenes, updates the client-side minimongo collection as soon as the server collection is updated.

<h3>Updating and Removing Map Markers in Realtime</h3>
Now it's time to add in the ```cursor.observe()``` ```changed()``` callback to update the party's icon, and ```removed()```  callback to delete the marker from the map and from the local ```markers``` hash.

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

Up to this point, there's been no visual indication _on the map_ as to which party is currently selected. Just like in the original Parties example, we solve this by creating a 50px x 50px transparent grey circular marker and making it concentric with the currently selected party's marker such that it forms a light halo around the selected party. The halo marker is purely a UI artefact that does not need to be saved on the server.

``` js
L.divIcon({
  iconSize: [50, 50], // set to 50px x 50px
  className: 'leaflet-animated-div-icon'
}
```
``` css
.leaflet-animated-div-icon {
  border-radius: 50%;
  border: none;
  opacity: .2;
  background: black;
}
```

<h3>Animating the Halo Marker</h3>
For a final flourish, we use the AnimatedMarker Leaflet plugin created by []() from OpenPlans to animate the marker as it moves about when a user selects different parties (rather than simply making it reappear at a different location). AnimatedMarker takes a Leaflet ```polyline``` object as the first argument to its  initialize function, and draws the marker at the beginning of the polyline. It then animates the marker along the polyline at a speed (in meters/ms) that's configurable via a second options argument. 

We need to make a minor tweak to the plugin's source code to support our needs: The original AnimatedMarker does not allow setting of the animation polyline after the marker is initialized. As a result, it requires the complete animation path to be known before creating the marker. But we want to create the AnimatedMarker once and to then set the animation polyline dynamically as soon as a user selects a different marker. The next animation polyline would be a single segment from the AnimatedMarker's current location to the center of the next marker selected. To accomplish this, I've had to reset the internal index (```this._i```) to 1 when a ```polyline``` is passed to the setLine method, making the method repeatedly reusable, rather than something that's called just once upon the marker's initialization. Until my pull request is merged, this modification is available at my fork on github.

[Ta-da!](http://chicago-parties.meteor.com)
