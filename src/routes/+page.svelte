<script>
	import { onMount } from 'svelte';
	import { MapboxOverlay } from '@deck.gl/mapbox';
	import { TripsLayer } from '@deck.gl/geo-layers';
	import { DataFilterExtension } from '@deck.gl/extensions';
	import { IconLayer } from '@deck.gl/layers';
	import maplibregl from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	import trips from './routes.json';
	import stops from './stops.json';
	import summary from './summary.json';

	delete summary.date;

	import color10 from './colors-tableau10.json';
	import color20 from './colors-tableau20.json';
	import pickupImg from '$lib/images/pin.png';
	import dropoffImg from '$lib/images/dest.png';
	import depotImg from '$lib/images/depot.png';

	let darkMode = false;
	const iconDefaultColor = darkMode ? [255, 255, 255] : [0, 0, 0];
	const mapStyle = darkMode
		? 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json'
		: 'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json';
	let colors;
	if (rawSummary.num_depots <= 10) {
		colors = color10;
	} else if (rawSummary.num_depots <= 20) {
		colors = color20;
	} else {
		throw new Error('Number of depots is too large, not enough colors to cover');
	}

	const startTime = 60 * 6; // 6:00
	const endTime = 60 * 22; // 22:00
	$: time = startTime;
	$: renderLayers(time);
	$: if (time >= endTime) time = startTime;

	let moving = false;
	let moveId = runVehicles();


	let currentVehicleId = 0;
	$: currentVehicle = trips.filter((trip) => trip.vehicle === currentVehicleId)[0];
	$: currentVehicleDepot = truncateLatLng(currentVehicle.original_path[0]);
	let map = null;
	let overlay = null;

	onMount(async () => {
		map = new maplibregl.Map({
			container: 'map', // container id
			style: mapStyle, // stylesheet location
			center: [114.107216, 22.352711], // starting position [lng, lat]
			zoom: 11 // starting zoom
		});
		map.addControl(new maplibregl.NavigationControl(), 'top-right');

		// Create the deck.gl overlay
		overlay = new MapboxOverlay({
			layers: [createTripLayer(0), createDepotIconLayer()],
			getTooltip: ({ object }) =>
				object &&
				`${object.type} 
        request: ${object.request_id}
        time: ${minutesToTime(object.time_window[0])} - ${minutesToTime(object.time_window[1])}
        num_passengers: ${object.num_passengers}
        num_wheelchairs: ${object.num_wheelchairs}`
		});
		map.addControl(overlay);
	});

	function renderLayers(time) {
		// https://deck.gl/docs/api-reference/mapbox/mapbox-overlay
		if (!overlay) return;

		overlay.setProps({
			layers: [
				createTripLayer(time),
				createPickupIconLayer(time),
				createDropoffIconLayer(time),
				createDepotIconLayer()
			]
		});
	}

	function createTripLayer(time) {
		return new TripsLayer({
			id: 'trips',
			data: trips,
			getPath: (d) => d.path,
			getTimestamps: (d) => d.time,
			getColor: (d) => colors[d.vehicle % summary.num_depots],
			currentTime: time,
			trailLength: 10,
			capRounded: true,
			jointRounded: true,
			widthMinPixels: 4
		});
	}

	const iconLayerDefaultProps = {
		data: stops,
		getPosition: (d) => d.coordinates,
		getSize: () => 20,
		getColor: () => iconDefaultColor,
		getFilterCategory: (d) => d.type,
		pickable: true
	};

	function createPickupIconLayer(time) {
		return new IconLayer({
			...iconLayerDefaultProps,
			id: 'pickup-icon',
			// iconAtlas:
			// 	'https://raw.githubusercontent.com/visgl/deck.gl/9.0-release/examples/website/icon/data/location-icon-atlas.png',
			// iconMapping:
			// 	'https://raw.githubusercontent.com/visgl/deck.gl/9.0-release/examples/website/icon/data/location-icon-mapping.json',
			// getIcon: (d) => getIconName(d.num_wheelchairs + d.num_passengers),
			getIcon: getIconFn(pickupImg),
			// show for 30 unit time before latest pickup time,
			getFilterValue: (d) => d.time_window[1],
			filterRange: [time, time + 30],
			filterCategories: ['pickup'],
			extensions: [new DataFilterExtension({ filterSize: 1, categorySize: 1 })]
		});
	}

	function createDropoffIconLayer(time) {
		return new IconLayer({
			...iconLayerDefaultProps,
			id: 'dropoff-icon',
			getIcon: getIconFn(dropoffImg),
			// show 30 unit time before latest dropoff time,
			getFilterValue: (d) => d.time_window[1],
			filterRange: [time, time + 30],
			filterCategories: ['dropoff'],
			extensions: [new DataFilterExtension({ filterSize: 1, categorySize: 1 })]
		});
	}

	function createDepotIconLayer() {
		return new IconLayer({
			...iconLayerDefaultProps,
			id: 'depot-icon',
			getIcon: getIconFn(depotImg),
			getColor: (d) => colors[getDepotNode(d.idx)],
			filterCategories: ['depot'],
			extensions: [new DataFilterExtension({ categorySize: 1 })]
		});
	}

	function resetTime() {
		// if there is any other side effect that is not defined in Deck.gl
		// and if they are dependent on time,
		// they need to be reset here
		time = startTime;
	}

	function runVehicles() {
		if (moving) return;

		moving = true;
		return setInterval(() => {
			time += 1;
		}, 50);
	}

	function stopVehicles(moveId) {
		if (moving) {
			moving = false;
			clearInterval(moveId);
		}
	}

	const getIconFn = (url) => () => ({
		url,
		width: 124,
		height: 124,
		mask: true
	});

	const getDepotNode = (stop_idx) => (stop_idx - summary.num_requests * 2) % summary.num_depots;

	function minutesToTime(mins) {
		mins = mins.toFixed(0);
		const hours = Math.floor(mins / 60);
		const minutes = mins % 60;
		return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`;
	}

	function truncateLatLng(latlng) {
		function truncate(num) {
			return Number(num).toFixed(6);
		}
		const [lat, lng] = latlng;
		return [truncate(lat), truncate(lng)];
	}

	function getGoogleMapUrl(lat, lng) {
		return `https://www.google.com/maps/search/?api=1&query=${lat},${lng}`;
	}
</script>

<main>
	<div id="map"></div>
	<input
		class="slider"
		type="range"
		min={startTime}
		max={endTime}
		step="1"
		bind:value={time}
		on:mousedown={() => stopVehicles(moveId)}
	/>
	<div class="datetime">
		<div class="date">2024-08-01</div>
		<div class="time">
			{minutesToTime(time)}
		</div>
		{#if moving}
			<button on:click={() => stopVehicles(moveId)}>Stop</button>
		{:else}
			<button on:click={() => (moveId = runVehicles())}>Run</button>
		{/if}
	</div>
	<figure class="summary">
		<figcaption>Summary Statistics</figcaption>
		<ul>
			{#each Object.entries(summary) as [key, value]}
				{#if key.includes('duration') || key.includes('time')}
					<li>{key.replaceAll('_', ' ')}: {value} mins</li>
				{:else}
					<li>{key.replaceAll('_', ' ')}: {value}</li>
				{/if}
			{/each}
		</ul>
	</figure>
	<select class="vehicle-select" bind:value={currentVehicleId}>
		{#each trips as trip}
			<option value={trip.vehicle}>Route of vehicle {trip.vehicle}</option>
		{/each}
	</select>
	<figure>
		<figcaption>Details of Vehicle {currentVehicleId}</figcaption>
		<div class="detail-wrapper">
			<figure class="summary">
				<figcaption>Summary</figcaption>
				<ul>
					{#each Object.entries(currentVehicle) as [key, value]}
						{#if !['path', 'original_path', 'time', 'original_time', 'vehicle'].includes(key)}
							{#if key.includes('duration') || key.includes('time')}
								<li>{key.replaceAll('_', ' ')}: {value} mins</li>
							{:else}
								<li>{key.replaceAll('_', ' ')}: {value}</li>
							{/if}
						{/if}
					{/each}
					<li>
						Depot: <a href={getGoogleMapUrl(currentVehicleDepot[1], currentVehicleDepot[0])}
							>{currentVehicleDepot[1]}, {currentVehicleDepot[0]}</a
						>
					</li>
				</ul>
			</figure>
			<figure class="route">
				<figcaption>Route</figcaption>
				<ol>
					{#each currentVehicle.original_path.map( (e, i) => [e, currentVehicle.original_time[i]] ) as [item, time]}
						{@const [lng, lat] = truncateLatLng(item)}
						<li>
							at {minutesToTime(time)}:
							<a href={getGoogleMapUrl(lat, lng)}>{lat}, {lng}</a>
						</li>
					{/each}
				</ol>
			</figure>
		</div>
	</figure>
</main>

<style>
	main {
		margin: 0 32px;
	}

	#map {
		width: 100%;
		height: 75vh;
	}
	.slider {
		width: 100%;
	}
	/*
	.embedded-text {
		line-height: 1;
		text-shadow: 2px 2px 4px black;
		background: rgba(255, 255, 255, 0.5);
	}
  */

	.datetime {
		display: flex;
		justify-content: space-between;
		gap: 16px;
		font-size: 48px;
	}
	.datetime > * {
		flex-grow: 1;
		flex-basis: 0;
	}
	.datetime > div:nth-child(2) {
		text-align: center;
	}
	.datetime > div:nth-child(3) {
		text-align: right;
	}

	figure {
		margin: 16px 0;
		font-size: 24px;
	}
	figure.summary > ul {
		columns: 2;
	}

	.vehicle-select {
		width: 100%;
		font-size: 24px;
	}
	.detail-wrapper {
		display: flex;
		gap: 32px;
	}
</style>
