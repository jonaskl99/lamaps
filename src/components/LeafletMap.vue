<template>
  <div id="map"/>
</template>

<script setup>
import {onMounted, ref} from "vue";
import L from 'leaflet'
import "leaflet/dist/leaflet.css";
import 'leaflet-textpath';

const initialMap = ref(null);
const orm = ref(null);
const ormSpeed = ref(null);
const laLayer = ref(null);
let la = {};

onMounted(() => {
  initialMap.value = L.map('map', {
    center: [52.172841433619034, 9.798002763342472],
    zoom: 12
  });

  //Basemap
  L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenStreetMap contributors',
  }).addTo(initialMap.value);

  //OpenRailwayMap
  orm.value = L.tileLayer('https://{s}.tiles.openrailwaymap.org/standard/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenRailwayMap contributors'
  }).addTo(initialMap.value);

  //Speed
  ormSpeed.value = L.tileLayer('https://{s}.tiles.openrailwaymap.org/maxspeed/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '&copy; OpenRailwayMap contributors'
  });

  //La
  laLayer.value = L.layerGroup().addTo(initialMap.value);

  //LayerMenu
  L.control.layers({}, {
    "Infrastruktur": orm.value,
    "Geschwindigkeiten": ormSpeed.value,
    "La-Strecken": laLayer.value,
  }).addTo(initialMap.value).expand();

  //Listeners
  loadData();
  initialMap.value.on('moveend', loadData)
});

async function loadData() {
  let bound = Object.values(initialMap.value.getBounds());
  let bbound = Object.values(bound[0]) + ',' + Object.values(bound[1]);

  const payload = "data=" + encodeURIComponent(
      '[out:json][bbox:' + bbound + '];way["ref:la"][railway=rail];out geom;'
  )
  await fetch("https://overpass-api.de/api/interpreter", {
    method: "POST",
    body: payload
  })
      .then(response => response.json())
      .then(function (data) {
        la = {};
        data.elements.forEach(element => {
              var latlngs = [];
              element.geometry.forEach(l => {
                latlngs.push(Object.values(l))
              })
              var polyline = L.polyline(latlngs, {
                opacity: 0,
                color: 'grey'
              }).setText('La ' + element.tags['ref:la'], {
                below: true,
                attributes: {fill: 'black', 'font-weight': 'italic', 'font-size': 20}
              })

              if (!(element.tags['ref:la'] in la)) {
                la[element.tags['ref:la']] = [];
              }
              la[element.tags['ref:la']].push(polyline);
            }
        );

        console.log(la)
        for (var i in la) {
          la[i].forEach(item => {
            item.addTo(laLayer.value);
          })
        }
      });
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