<title>Bikewatching</title>
<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";

    mapboxgl.accessToken = "pk.eyJ1IjoiYWNldmVkb2pldHRlciIsImEiOiJjbTk1cmIxcncxZGNsMmxwbnhwd2FmdTJmIn0.fgwQcMBzGMXWSPyXL1p7Gg";
    let map;
    let stations = [];
    let mapViewChanged = 0;
    let trips = [];
    let departures;
    let arrivals;
    let timeFilter = -1;
    let isochrone = null;
    let selectedStation = null;

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    async function initMap() {
        map = new mapboxgl.Map({
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
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });        
    }
    
    onMount(() => {
        initMap();
    })

    $: map?.on("move", evt => mapViewChanged++);
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range([3, 30]);

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"});
    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });
    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
    $: filteredStations = stations.map(station => {
        const id = station.Number;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return {
            ...station,
            arrivals: arr,
            departures: dep,
            totalTraffic: arr + dep
        };
    });

    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
    const profile = "cycling";
    const minutes = [5, 10, 15, 20];
    const contourColors = [
        "03045e",
        "0077b6",
        "00b4d8",
        "90e0ef"
    ]
    async function getIso(lon, lat) {
        const base = `${urlBase}${profile}/${lon},${lat}`;
        const params = new URLSearchParams({
            contours_minutes: minutes.join(","),
            contours_colors: contourColors.join(","),
            polygons: "true",
            access_token: mapboxgl.accessToken
        });
        const url = `${base}?${params.toString()}`;

        const query = await fetch(url, { method: 'GET' });
        isochrone = await query.json();
    }

    function geoJSONPolygonToPath(feature) {
        const path = d3.path();
        const rings = feature.geometry.coordinates;

        for (const ring of rings) {
            for (let i = 0; i < ring.length; i++) {
                const [lng, lat] = ring[i];
                const { x, y } = map.project([lng, lat]);
                if (i === 0) path.moveTo(x, y);
                else path.lineTo(x, y);
            }
            path.closePath();
        }
        return path.toString();
    }
    
    $: if (selectedStation) {
        getIso(+selectedStation.Long, +selectedStation.Lat);
    } else {
        isochrone = null;
    }
</script>
<header>
    <h1>Welcome to bikewatch</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>

<p>This is the brief description of what it does</p>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                            d={geoJSONPolygonToPath(feature)}
                            fill={feature.properties.fillColor}
                            fill-opacity="0.2"
                            stroke="#000000"
                            stroke-opacity="0.5"
                            stroke-width="1"
                    >
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
                <circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} class={station?.Number === selectedStation?.Number ? "selected" : ""}
                    on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}
                    style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }" fill="steelblue" fill-opacity="60%" stroke="white" pointer-events="auto">
                    <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                </circle>
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
	<div class="departures" style="--departure-ratio: 1">More departures</div>
	<div class="balanced" style="--departure-ratio: 0.5">Balanced</div>
	<div class="arrivals" style="--departure-ratio: 0">More arrivals</div>
</div>
<style>
    @import url("$lib/global.css");

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
        margin: auto;
    }

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
        &:has(circle.selected) circle:not(.selected) {
            opacity: 0.3;
        }
    }

    circle {
        transition: opacity 0.2s ease;
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }

    .legend {
        display: flex;
        margin-block: auto;
    }

    .departures, .balanced, .arrivals {
        flex: 1;
        margin: 0.1em;
        text-align: center;
        padding: 1em;
    }

    .departures {
        background-color: steelblue;
    }

    .balanced {
        background-color: color-mix(
            in oklch,
            steelblue calc(100% * var(--departure-ratio)),
            darkorange
        );
    }

    .arrivals {
        background-color: darkorange;
    }
</style>