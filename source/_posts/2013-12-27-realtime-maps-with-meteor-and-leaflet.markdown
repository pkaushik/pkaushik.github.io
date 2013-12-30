---
layout: post
title: "Realtime Maps with Meteor and Leaflet - Part One"
date: 2013-12-27 20:51:10 -0600
comments: true
categories: [meteor, webapps, leaflet]
---
{% imgcap left https://raw.github.com/meteor/meteor/devel/examples/parties/public/soma.png 350 350 this 'map' is actually a static image %} 

The [parties example](https://www.meteor.com/examples/parties) bundled with [Meteor](http://www.meteor.com) is a nifty demonstration of the framework's core principles, but it uses a [500 x 500 pixel image of downtown San Francisco](https://github.com/meteor/meteor/blob/devel/examples/parties/public/soma.png) as a faux map. This means that we cannot pan or zoom the "map," and when we double-click the image to create new parties, the circle markers are drawn at the position of the clicks in relation to the _image element in the browser window_, and not at geospatial coordinates. 
<!--more-->
{% imgcap left /images/custom/old-parties.png 350 350 circles drawn over the static image %} 

I decided to update the example to use [Leaflet.js](http://leafletjs.com/) to make a real map that looked and felt as close to the original example as possible. In particular, I wanted to preserve the color-coded circles (red for private, blue for public parties) labeled with the number of RSVPs, and the larger animated circle indicating which party is currently selected, with its details displayed in a section outside the map. This is a useful pattern for displaying individual marker details without using a popup that occludes part of the map. 

Here is the [end result](http://chicago-parties.meteor.com) with [source code](https://github.com/pkaushik/parties). In the next two posts, I will go over the changes I made to the original example. I won't be covering how Meteor works, and will assume you have some understanding of how the parties example works as well.

<h3>Setting the Stage</h3>
First off, I created the example and added leaflet to the project using [Meteorite](http://oortcloud.github.io/meteorite/). 
```
$ meteor create --example parties

$ cd parties

$ mrt add leaflet
leaflet: Leaflet.js, mobile-friendly interactive maps....
```
I then edited the `page` template to use Bootstrap's fluid classes to generate a [responsive page layout](http://getbootstrap.com/2.3.2/scaffolding.html#responsive) and added a `window.resize()` handler to adjust the map's size as the browser is resized. I use this pattern when creating responsive Leaflet maps, and it's not specific to Meteor. 

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
[Stamen Design](http://stamen.com/)'s [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) make a nice replacement for the black & white map image in the example. I disabled double-click and touch zoom when initializing the map since those actions are how users create new parties, and I increased tile opacity to lighten the overall background and improve the visibility of markers on the map. Leaflet initialization code goes into the `map` template's `rendered()` callback. 

``` js
map = L.map($('#map_canvas'), {
  doubleClickZoom: false,
  touchZoom: false
}).setView(new L.LatLng(41.8781136, -87.66677956445312), 13);

L.tileLayer('http://{s}.tile.stamen.com/toner/{z}/{x}/{y}.png', {opacity: .5}).addTo(map);
```
The next significant change was to replace the `map` template's event handler from the original example with Leaflet's `"dblclick"` event handler to manage the creation of new parties. The Leaflet version conveniently returns a `LatLng` which I saved to a Session variable before triggering `createDialog`. The mechanism to trigger dialogs by setting the associated Session variables `Session.showCreateDialog` and `Session.showInviteDialog` is unchanged from the original example, and it works because Meteor Session variables are [reactive](http://docs.meteor.com/#reactivity).

``` js
map.on("dblclick", function(e) {
  if (! Meteor.userId()) // must be logged in to create parties
    return;
  
  Session.set("createCoords", e.latlng);
  Session.set("showCreateDialog", true);
});
```
``` html 
{% raw %}<template name="page">
  {{#if showCreateDialog}}
    {{> createDialog}}
  {{/if}}
  ...
  ...
</template>{% endraw %}
```
<h3>Creating and Saving a Party to the Database</h3>
This part of the application is also more or less unchanged from the original example except that I passed the party's `LatLng` (instead of click position) along with other details from the `createDialog` template to the `Meteor.methods()` call to `createParty`. If the callback is successful, the new party's `_id` is saved to another reactive Session variable `Session.selected`, which drives the `details` template on the left.
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
  if (! error) { //party was successfully added to the server's mongo collection
    Session.set("selected", partyId);
    ...
  }
});
```
<h3>Adding Markers to the Map in Realtime</h3>
As soon as a new party is added to the Parties mongo collection on the server, behind the scenes, Meteor transmits it back to a client-side minimongo collection with the same name on all connected and authorized clients. This can be verified by typing `Parties.findOne()` into the JavaScript console. This is well and good, but the next task is to replace the [D3](http://d3js.org) code to draw circles from the original example with code to add Leaflet markers to the map. 

To do that, I hooked up a `cursor.observe()` `added()` callback to create the map marker and I added a click handler to the marker to update the `Session.selected` variable with the party's `_id`. As users click on different parties, this reactively triggers the context for the `details` template on the left. I also saved a reference to the marker in a local `markers` hash to efficiently access the marker for future changes. Since we only need to set this up once, I put this code into the `map` template's `created()` callback.
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
  });
}
```
The final bit of fanciness here is my `createIcon()` helper function to create a lightweight `DivIcon` that uses a simple `div` element instead of an image icon. I used CSS `border-radius` to style the `div` as a circle of the appropriate color and set CSS `line-height` to the height of the `div` to vertically center the text. The `attending()` helper function from the original example returns the number of Yes RSVPs. 
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
``` js
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

.leaflet-div-icon.public { 
  background: #49AFCD; 
}

.leaflet-div-icon.private { 
  background: #DA4F49; 
}
```
Now I can log in and create a few parties, and they all show up as markers with the appropriate color and label. When I click on a marker, its details are automatically rendered into the `details` template on the left. But there's no visual indication _on the map_ as to which party is currently selected -- I just need to remember which marker I clicked on last! As it turns out, this usability quirk is easy to address.

[Part Two: Updating and deleting parties, and animating the selected party indicator...]({{root_dir}}/blog/2013/12/28/realtime-maps-with-meteor-and-leaflet-part-2/)

