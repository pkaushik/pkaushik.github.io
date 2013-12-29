---
layout: post
title: "Realtime Maps with Meteor and Leaflet - Part One"
date: 2013-12-27 20:51:10 -0600
comments: true
categories: [meteor, webapps, leaflet]
---

{% imgcap left https://raw.github.com/meteor/meteor/devel/examples/parties/public/soma.png 350 350 this 'map' is actually a static image %} 
The [parties example](https://www.meteor.com/examples/parties) bundled with [Meteor](http://www.meteor.com) is a nifty demonstration of the framework's core principles, but it uses a [500 x 500 pixel image of downtown San Francisco](https://github.com/meteor/meteor/blob/devel/examples/parties/public/soma.png) as a faux map. This means that we cannot pan or zoom the "map" and when we double-click the image to create new parties, the circle markers are drawn at the position of the clicks in relation to the _image element in the browser window_, and not at geospatial coordinates. 

{% imgcap left /images/custom/old-parties.png 350 350 circles drawn over the static image %} 

I decided to update the example to use [Leaflet.js](http://leafletjs.com/) to make a real map that looks and feels as close to the original example as possible. In particular, I wanted to preserve the color-coded circles (red for private, blue for public parties) labeled with the number of RSVPs, and the larger animated circle indicating which party is currently selected, with its details displayed in the sidebar. This is a useful pattern for displaying individual marker details without using a popup that occludes part of the map. Here's the [end result](http://chicago-parties.meteor.com) with [source code](https://github.com/pkaushik/parties).

Over the next two posts, I'll go over the changes I made to the original parties example. I won't explain how Meteor works, and it would help if you've run the original example and have a rough sense of what's going on.

<h3>Setting the Stage</h3>
First off, we create the original example and add leaflet to the project using the [meteorite](http://oortcloud.github.io/meteorite/) command line. Meteorite is Meteor's unofficial package management system. 
```
$ meteor create --example parties
parties: created.

To run your new app:
   cd parties
   meteor

$ cd parties

$ mrt add leaflet
leaflet: Leaflet.js, mobile-friendly interactive maps....
```

Since we'll no longer be calculating relative coordinates to draw circles on a jpeg image, the base map does not need to be a fixed size. We can use Bootstrap's fluid classes to generate a [responsive page layout](http://getbootstrap.com/2.3.2/scaffolding.html#responsive) and adjust the map's size as the browser is resized. This pattern is frequently used when creating responsive Leaflet maps, and not specific to Meteor. 

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
Leaflet initialization code goes into the map template's ```rendered()``` callback. [Stamen Design](http://stamen.com/)'s [toner themed map tiles](http://maps.stamen.com/toner/#12/37.7706/-122.3782) are a nice replacement for the black & white map image in the original example. Disabling click and touch zoom onthe map lets us reserve those actions for users to create a new party, and increasing the map tiles' opacity improves the visibility of markers against the very dark toner theme. 

``` js
map = L.map($('#map_canvas'), {
  doubleClickZoom: false,
  touchZoom: false
}).setView(new L.LatLng(41.8781136, -87.66677956445312), 13);

L.tileLayer('http://{s}.tile.stamen.com/toner/{z}/{x}/{y}.png', {opacity: .5}).addTo(map);
```

Next we replace the map template's event handler from the original example with Leaflet's ```"dblclick"``` event handler which conveniently returns a Leaflet ```LatLng``` with its callback, which we promptly save as the Meteor Session variable ```Session.createCoords```. Because Meteor Session variables are reactive, setting ```Session.showCreateDialog``` triggers the display of a dialog allowing the user to enter additional details about the new party. This mechanism is unchanged from the original example.

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
Clicking the create dialog's 'Save' button makes a ```Meteor.methods()``` call that is virtually the same as in the original example except that the method now takes the ```LatLng``` that we just saved as ```Session.createCoords```. When the new party has been successfully added to a server-side mongo collection called Parties, we can save the new party's ```_id``` to the ```Session.selected``` variable. 

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
    ...
  }
});
```

<h3>Adding Markers to the Map in Realtime</h3>
So far, we've saved a newly created party into a server-side mongo collection and Meteor, behind the scenes, has transmitted it back to a client-side minimongo collection with the same name (Parties). We can verify this by typing ```Parties.findOne()``` into the JavaScript console. 

Let's now observe changes to a client-side query of the minimongo collection, and automatically draw a map marker whenever a new party is added. Since this only needs to be set up once, we can put the ```cursor.observe()``` code in the map template's ```created()``` callback. This replaces the D3 svg circle drawing code from the original example.

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
In the ```added()``` callback, we create a new marker, add it to the map, and set up a click handler to store the associated party's ```_id``` in another Meteor Session variable, ```Session.selected```. We also store a reference to the marker in a locally scoped hash keyed by the associated party's ```_id``` to efficiently update / delete markers when party details change. 

The ```createIcon``` function returns a lightweight [```DivIcon```](http://leafletjs.com/reference.html#divicon) that uses a simple ```div``` element instead of an image, and CSS ```border-radius``` to style the div as a circle of the appropriate color. The ```attending()``` helper method from the original example returns the number of Yes RSVPs. CSS ```vertical-align``` and ```line-height``` set to the height of the ```div``` when used in combination ensure the HTML text is vertically centered.

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

.leaflet-div-icon.public { 
  background: #49AFCD; 
}

.leaflet-div-icon.private { 
  background: #DA4F49; 
}
```
At this point, we can create a bunch of parties and they all show up as markers with the appropriate color and label. When we click on a marker, ```Session.selected``` is updated by the marker's click handler. Because Meteor Session variables are [reactive](http://docs.meteor.com/#reactivity), this automatically renders the details template in the sidebar. 

But there's no visual indication _on the map_ as to which party is currently selected -- we just need to remember which marker we clicked on last! As it turns out, this usability issue is easy to fix and we'll see how in the next post.

[Part Two: Updating and deleting parties, and animating the selected party indicator...]({{root_dir}}/blog/2013/12/28/realtime-maps-with-meteor-and-leaflet-part-2/)

