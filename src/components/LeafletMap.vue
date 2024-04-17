<template>
  <div id="map"/>
</template>

<script setup>
import {onMounted} from "vue";
import L from 'leaflet'
import "leaflet/dist/leaflet.css";
import 'leaflet-textpath';

let initialMap = null;
let orm = null;
let ormSpeed = null;
let laLayer = null;
let elements = [];

onMounted(() => {
  initialMap = L.map('map', {
    center: [51.0, 8.0],
    zoom: 7
  });

  //Basemap
  L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenStreetMap contributors',
  }).addTo(initialMap);

  //OpenRailwayMap
  orm = L.tileLayer('https://{s}.tiles.openrailwaymap.org/standard/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenRailwayMap contributors'
  }).addTo(initialMap);

  //Speed
  ormSpeed = L.tileLayer('https://{s}.tiles.openrailwaymap.org/maxspeed/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenRailwayMap contributors'
  });

  //La
  laLayer = L.layerGroup().addTo(initialMap);

  //LayerMenu
  L.control.layers({}, {
    "Infrastruktur": orm,
    "Geschwindigkeiten": ormSpeed,
    "La-Strecken": laLayer,
  }).addTo(initialMap).expand();

  //Listeners
  loadData();
  initialMap.on('moveend', loadData)
});

function loadData() {
  if (initialMap.getZoom() < 12) {
    console.log('out of zoom');
    return;
  }

  let bound = Object.values(initialMap.getBounds());
  let bbound = Object.values(bound[0]) + ',' + Object.values(bound[1]);

  const payload = "data=" + encodeURIComponent(
      '[out:json][timeout:5];(way["railway"]["ref:La"](' + bbound + ');way["railway"]["ref:la"](' + bbound + '););out geom;'
  )
  fetch("https://overpass-api.de/api/interpreter", {
    method: "POST",
    body: payload
  })
      .then(response => response.json())
      .then(function (data) {
        if (!data.elements) {
          return;
        }
        data.elements.forEach(element => {
          if (elements.includes(element.id)) {
            return;
          }
          elements.push(element.id)
          let latlngs = [];
          let laNumber = 0;
          if (element.tags['ref:La'] === undefined) {
            laNumber = element.tags['ref:la'];
          } else {
            laNumber = element.tags['ref:La']
          }

          let color = genColorFromAttribute(laNumber);
          element.geometry.forEach(geom => {
            latlngs.push([geom.lat, geom.lon]);
          });
          L.polyline(latlngs, {color: color})
              .bindPopup('La- Strecke '+laNumber)
              .addTo(laLayer)
        })
      });
}

function genColorFromAttribute(attribute) {
  let hash = 0;
  for (let i = 0; i < attribute.length; i++) {
    hash = attribute.charCodeAt(i) + ((hash << 5) - hash);
  }
  let c = (hash & 0x00FFFFFF)
      .toString(16)
      .toUpperCase();
  return '#' + "00000".substring(0, 6 - c.length) + c;

}
</script>
<style>
body {
  margin: 0;
}

#map {
  height: 100vh;
  width: 100vw;
}
</style>