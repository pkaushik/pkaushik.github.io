---
layout: page
date: 2012-05-22 14:25
comments: true
sharing: true
footer: true
---
Pallavi Anderson lives in Chicago's West Loop with her husband, having previously resided in Boston, Bangalore, Bombay, Riyadh and Dammam. She enjoys global travel, international cuisine, and the occasional Thinkgeek puzzle. 

An alumnus of the [MIT Media Lab](http://www.media.mit.edu/research/groups/changing-places) with a prior degree in Architecture, she has conducted research on urban redevelopment & transportation infrastructure in Bombay, [aging-in-place & assistive (“smart”) home technologies](http://architecture.mit.edu/house_n/placelab.html), developed a bilingual mobile application to support monitoring TB patients in rural India, and has previously been featured on [Discovery Channel Canada](http://architecture.mit.edu/house_n/videos/PlaceLabDiscoveryChannel04.asf) (asf movie clip) & [Slashdot](http://science.slashdot.org/science/08/02/09/1825201.shtml). One of her primary interests is figuring out how to broaden the reach of technology to all members of society, particularly those who are part of under-served communities. She recently led a team of Chicago students to build an English-Spanish web and smartphone application to help the residents of Chicago's Little Village participate in the planning, design and use of their new park.

Statements and opinions on this site are her own, and do not necessarily represent positions, strategies, or opinions of her present or past employers.

<link rel="stylesheet" href="http://cdn.leafletjs.com/leaflet-0.7.1/leaflet.css" />
<script type="text/javascript" src="http://cdn.leafletjs.com/leaflet-0.7.1/leaflet.js"></script>
<link rel="stylesheet" href="/javascripts/custom/leaflet-label.css" />
<script type="text/javascript" src="/javascripts/custom/leaflet-label.js"></script>

<div id="map"></div>

<script type="text/javascript">
  var map = L.map('map').setView([30.8, 0], 2);
  L.tileLayer('http://{s}.tile.stamen.com/watercolor/{z}/{x}/{y}.jpg', {
      maxZoom: 18,
      minZoom: 2
  }).addTo(map);
  
  map.attributionControl.setPrefix(''); 
	var attribution = new L.Control.Attribution();
  attribution.addAttribution("Geocoding data &copy; 2013 <a href='http://open.mapquestapi.com'>MapQuest, Inc.</a>");
  attribution.addAttribution("Map tiles by <a href='http://stamen.com'>Stamen Design</a> under <a href='http://creativecommons.org/licenses/by/3.0'>CC BY 3.0</a>.");
  attribution.addAttribution("Data by <a href='http://openstreetmap.org'>OpenStreetMap</a> under <a href='http://creativecommons.org/licenses/by-sa/3.0'>CC BY SA</a>.");
  map.addControl(attribution);
  
  var livedIn = [{
      city: "Bombay", population: "20.5M", livedin: "all the rest", 
      coords: [18.9750, 72.8258], labelDir: "left", labelAnchor: [-6, 10]
    }, {
      city: "Riyadh", population: "5.2M", livedin: "1981-1985", 
      coords: [24.6333, 46.7167], labelDir: "left", labelAnchor: [-6, -10]
    }, {
      city: "Dammam", population: "2M", livedin: "1987-1990", 
      coords: [26.2833, 50.2000], labelDir: "right", labelAnchor: [-6, -10]
    }, {
      city: "Bangalore", population: "8.5M", livedin: "1998-2003", 
      coords: [12.9667, 77.5667], labelDir: "right", labelAnchor: [-6, 10]
    }, {
      city: "Boston", population: "2.5M", livedin: "2003-2005", 
      coords: [42.3581, -71.0636], labelDir: "right", labelAnchor: [-6, 10]
    }, {
      city: "Chicago", population: "8.7M", livedin: "2005-now", 
      coords: [41.8819, -87.6278], labelDir: "left", labelAnchor: [-6, 10]
    }
  ]
  
  for (var i = 0; i < livedIn.length; i++) {
    var l = livedIn[i];
    var marker = L.marker(l.coords, {
      icon: L.divIcon({
        iconSize: [10, 10],
        className: "leaflet-div-icon",
        labelAnchor: l.labelAnchor
      })
    }).addTo(map);
    
    marker.bindLabel(/*l.city + "<br>" + l.population + "<br>" + */l.livedin, {
      direction: l.labelDir,
      noHide: true
    }).showLabel();
  }
  
  
</script>