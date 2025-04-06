<title>Bikewatching</title>
<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";

    mapboxgl.accessToken = "pk.eyJ1IjoiYWNldmVkb2pldHRlciIsImEiOiJjbTk1cmIxcncxZGNsMmxwbnhwd2FmdTJmIn0.fgwQcMBzGMXWSPyXL1p7Gg";
    let stations = [];
    let map;

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    async function initMap() {
        let map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/light-v11",
            center: [-71.09147010427712, 42.35887342870736],
            zoom: 12,
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addLayer({
            id: "boston bikelanes",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.5,
            },
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
            id: "cambridge bikelanes",
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.5,
            },
        });

        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    }

    onMount(() => {
        initMap();
    })
</script>
<h1>Welcome to Bikewatching</h1>
<p>This is the brief description of what it does</p>
<div id="map">
    <svg>
        {#each stations as station}
            <circle { ...getCoords(station) } r="5" fill="steelblue" />
        {/each}
    
    </svg>
</div>
<style>
    @import url("$lib/global.css");

    #map {
        flex: 1;
        background-color: pink;
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%; 
        height: 100%;
        pointer-events: none;
    }
</style>