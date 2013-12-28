---
layout: post
title: "Realtime Maps with Meteor and Leaflet - Part 1"
date: 2013-12-27 20:51:10 -0600
comments: true
categories: [meteor, webapps, leaflet]
---

{% imgcap left https://raw.github.com/meteor/meteor/devel/examples/parties/public/soma.png 350 350 the map is actually a static image %} 
The [Parties example](https://www.meteor.com/examples/parties) bundled with [Meteor](http://www.meteor.com) is a nifty demonstration of the framework's core principles. It also shows how we might implement fine-grained data access control to limit who can see and edit specific items in a collection - in this case which users are allowed to RSVP to parties. But the application isn't nearly as useful as it could be because it displays a [500 x 500 pixel image of downtown San Francisco](https://github.com/meteor/meteor/blob/devel/examples/parties/public/soma.png) as the faux map. This means that we cannot pan or zoom the "map," nor can we initialize it to any other location or zoom level, and when we double click the image to create a party, D3 draws a circle to mark the position of the click in relation to the _image element in the browser window_, and not at geospatial coordinates. 

I decided to refactor the example to use [Leaflet.js](http://leafletjs.com/) to make a real map that looks and feels as close to the original as possible. The color-coded circle markers (pink for private, blue for public parties) labeled with the number of RSVPs are nice and lightweight, and I like how the larger animated circle is used to indicate which party is currently selected, with details about the party shown in the sidebar. It's a useful pattern for showing details about an individual map marker without using the standard popup approach which occludes part of the map.

TLDR; End result: [http://chicago-parties.meteor.com](http://chicago-parties.meteor.com) and [source code for the complete application](https://github.com/pkaushik/parties). You need a github account to log in, to create or RSVP to parties.

<h3>Setting the Stage</h3>

First off, since we'll no longer be calculating relative x and y coordinates to position circles on a jpeg image, the map does not need to be of a fixed size. We modify the page template to use Bootstrap's fluid classes to generate a responsive layout, and include the following boilerplate to adjust the map's size as the window is resized. This pattern is standard practice when creating responsive Leaflet maps; not specific to Meteor. 

``` html
{% raw %}<div class="container-fluid">
  <div class="row-fluid">
    <div class="span4">
      {{> details}}
      {{#if currentUser}}
      <div class="pagination-centered">
        <em><small>Double click the map to post a party!</small></em>
      </div>
      {{/if}}
    </div>
    <div class="span8">
        {{> map}}
    </div>
  </div>
</div>{% endraw %}
```

``` js
$(window).resize(function () {
  var h = $(window).height(), offsetTop = 90; // Calculate the top offset
  $mc = $('#map_canvas');
  $mc.css('height', (h - offsetTop));
}).resize();
```

<h3>Map Initialization</h3>
Map initialization code goes into the template's ```rendered()``` callback. Stamen's [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) are a nice replacement for the black & white map image. Disabling click and touch zoom lets us reserve those actions for creating a new party, and adjusting the base map's opacity improves the visibility of markers against the very dark toner theme. 

``` js
map = L.map($('#map_canvas'), {
  doubleClickZoom: false,
  touchZoom: false
}).setView(new L.LatLng(41.8781136, -87.66677956445312), 13);

L.tileLayer('http://{s}.tile.stamen.com/toner/{z}/{x}/{y}.png', {opacity: .5}).addTo(map);
```

Next we replace the map template's event handler from the original example with Leaflet's ```"dblclick"``` event handler which conveniently returns a Leaflet ```LatLng``` as the argument to its callback.

``` js
map.on("dblclick", function(e) {
  if (! Meteor.userId()) // must be logged in to create parties
    return;
  
  Session.set("createCoords", e.latlng);
  ...
  ...
});
```

<h3>Creating and Saving a Party to the Database</h3>
The code to create a new party using a ```Meteor.methods()``` call is virtually the same as in the original example except that the method now takes the ```LatLng``` that we saved as ```Session.createCoords```. When the new party has been created and successfully saved in a server-side mongo collection, we get a successful callback and save the new party's ```_id``` to the ```Session.selected``` variable. This ensures that the most recently created party is always selected.

``` js
var title = template.find(".title").value;
var description = template.find(".description").value;
var public = ! template.find(".private").checked;
var latlng = Session.get("createCoords");
   
Meteor.call('createParty', {
  title: title,
  description: description,
  latlng: latlng,
  public: public
}, function (error, partyId) {
  if (! error) {
    Session.set("selected", partyId);
    ...
    ...
  }
});
```

<h3>Adding, Removing, and Updating Markers on the Map in Realtime</h3>
So far, we've saved a newly created party into the server-side Parties mongo collection and behind the scenes, Meteor has transmitted it back to the client-side monomongo collection with the same name. We can verify this by typing ```Parties.findOne()``` into the JavaScript console. 

Let's now observe changes to the client's monomongo collection and automatically draw a map marker every time a new party is added to the collection. Since this only needs to be set up once, we can put the following code in the map template's ```created()``` callback. This replaces the D3 circle drawing code from the original example.

``` js
var map, markers = {};

Template.map.created = function() {
  Parties.find({}).observe({
    added: function(party) {
      var marker = new L.Marker(party.latlng, {
        _id: party._id,
        icon: createIcon(party)
      }).on('click', function(e) {
        Session.set("selected", e.target.options._id);
      });      
      map.addLayer(marker);
      markers[marker.options._id] = marker;
    },
    ...
    ...
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
The ```added()``` callback stores a reference to the new marker in a hash keyed by ```Party._id```. This is useful since we need to efficiently update / delete markers when the RSVP count changes or a party is deleted. The ```createIcon``` function uses the new lightweight [```DivIcon```](http://leafletjs.com/reference.html#divicon) that uses a simple ```div``` element instead of an image and CSS ```border-radius``` to style the div as a circle of the appropriate color. The ```attending()``` from the original example returns the number of Yes RSVPs. CSS ```line-height``` (set tot he height of the ```div```) and ```vertical-align``` used in combination ensure the HTML text is vertically centered.

``` js
var createIcon = function(party) {
  var className = 'leaflet-div-icon ';
  className += party.public ? 'public' : 'private';
  return L.divIcon({
    iconSize: [30, 30], // set size to 30px x 30px
    html: '<b>' + attending(party) + '</b>',
    className: className  
  });
}
```
``` css
.leaflet-div-icon {
    border-radius: 50%;
    border: none;
    line-height: 30px; 
    font-family: verdana;
    text-align: center;
    color: white;
    opacity: .8;
    vertical-align: middle;
}

.leaflet-div-icon.public { background: #49AFCD; }

.leaflet-div-icon.private { background: #DA4F49; }
```

<em>Next: Updating and deleting parties, and animating the selected party indicator...</em>

