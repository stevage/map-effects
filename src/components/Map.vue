<template lang="pug">
    #map
</template>

<script>
import mapboxgl from 'mapbox-gl';
import U from 'mapbox-gl-utils';
const turf = require('@turf/turf');
export default {
    mounted() {
        mapboxgl.accessToken = 'pk.eyJ1Ijoic3RldmFnZSIsImEiOiJGcW03aExzIn0.QUkUmTGIO3gGt83HiRIjQw';
        const map = new mapboxgl.Map({
            container: 'map',
            center: [145, -37.8],
            zoom: 17,
            style: 'mapbox://styles/mapbox/light-v9',
            pitch:30
        });
        U.init(map);
        window.map = map;
        window.Map = this;
        // map.U.onLoad(e => startMap(map));
        map.U.onLoad(e => roadExtend(map));
    }
}
// const featureIds = new Set();
function roadExtend(map) {

    const roadBits = {};

    function computeRoadExtensions() {
        const features = [];
        Object.keys(roadBits).forEach(roadBitId => {
            const roadBit = roadBits[roadBitId];
            features.push(turf.lineSliceAlong(roadBit.feature, 0, roadBit.progress * roadBit.length));
            roadBit.progress = Math.min(roadBit.progress + 0.01, 1);
        });
        map.U.update('road-extension', turf.featureCollection(features));
        window.setTimeout(computeRoadExtensions, 50);
    }

    map.U.addGeoJSON('road-extension')
        .addLine('road-extension', { 
            lineColor: 'magenta',
            lineWidth: 10,
            lineOpacity: 0.7,
            lineBlur: 1
        });

    map.on('moveend', e => {
        const roads = map.querySourceFeatures('composite', { sourceLayer: 'road', filter: ['all', /*['in', 'class', 'motorway', 'trunk'],*/ ['==', '$type', 'LineString']] });
        roads.forEach(road => {
            const id = `${road.id}/${road.tile.x}/${road.tile.y}/${road.tile.z}`
            if (roadBits[id]) {
                return;
            }
            roadBits[id] = {
                length: turf.length(road),
                feature: turf.flatten(road).features[0], // just consider the first part of any multilinestring for now
                progress: 0.01
            }
            // turf.lineSliceAlong()
        });
    })
    window.setTimeout(computeRoadExtensions, 200);

}
function buildingsJiggle(map) {
    const featureIds = {};
    function updateBuildings() {
        Object.keys(featureIds).forEach(id => {
            featureIds[id] = Math.max(0, featureIds[id] + 5 - Math.random() * 10);
            map.setFeatureState({ id, source: 'composite', sourceLayer: 'building' }, { 
                h: featureIds[id] });
        });
        window.setTimeout(updateBuildings, 100);
    }
    map.U.addFillExtrusion('building-3d', 'composite', {
        sourceLayer: 'building',
        fillExtrusionColor: 'magenta',
        fillExtrusionHeight: ['to-number', ['feature-state', 'h'], ['get', 'height'], 50]
    });
    map.on('moveend',e => {
        const buildings = map.querySourceFeatures('composite', { sourceLayer: 'building' });
        buildings.forEach(b => featureIds[b.id] = featureIds[b.id] || 100);
        

    });
    map.on('sourcedata', e => {
        // console.log(e.tile);
    });
    window.setTimeout(updateBuildings, 500);
}

import 'mapbox-gl/dist/mapbox-gl.css';

</script>

<style scoped>
#map {
    width: 100%;
    height: 100%;
    position:absolute;
}
</style>