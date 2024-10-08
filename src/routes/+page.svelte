<script>
	import { onMount } from 'svelte';
	import { MapboxOverlay } from '@deck.gl/mapbox';
	import { TripsLayer } from '@deck.gl/geo-layers';
	import { DataFilterExtension } from '@deck.gl/extensions';
	import { IconLayer } from '@deck.gl/layers';
	import maplibregl from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	import trips from './routes.json';
	import stops from './stop_solutions.json';
	import rawSummary from './summary.json';

	import color10 from './colors-tableau10.json';
	import color20 from './colors-tableau20.json';
	import pickupImg from '$lib/images/pin.png';
	import dropoffImg from '$lib/images/dest.png';
	import depotImg from '$lib/images/depot.png';

	let colors;
	if (rawSummary.num_depots <= 10) {
		colors = color10;
	} else if (rawSummary.num_depots <= 20) {
		colors = color20;
	} else {
		throw new Error('Number of depots is too large, not enough colors to cover');
	}

	const date = rawSummary.date;
	// exclude 'date' keys
	const summary = Object.fromEntries(
		Object.entries(rawSummary).filter(([key, _]) => !key.includes('date'))
	);
	const serviceStartTime = 60 * 6; // 6:00
	const serviceEndTime = 60 * 22; // 22:00

	let map = null;
	let overlay = null;
	let darkMode = false;
	let showRequests = false;
	let moving = false;
	let moveId = runVehicles();
	let currentVehicle;

	$: mapStyle = darkMode
		? 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json'
		: 'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json';
	$: iconDefaultColor = darkMode ? [255, 255, 255] : [0, 0, 0];

	$: currentVehicleDepot = currentVehicle ? truncateLatLng(currentVehicle.original_path[0]) : null;

	$: startTime = currentVehicle ? currentVehicle.original_time[0] : serviceStartTime;
	$: endTime = currentVehicle
		? currentVehicle.original_time[currentVehicle.original_time.length - 1]
		: serviceEndTime;
	$: time = startTime;

	$: filteredTrips = currentVehicle ? [currentVehicle] : trips;
	$: filteredStops = currentVehicle
		? stops.filter((stop) => currentVehicle?.route.includes(stop.idx))
		: stops;

	$: iconLayerDefaultProps = {
		data: filteredStops,
		getPosition: (d) => d.coordinates,
		getSize: () => 20,
		getColor: () => iconDefaultColor,
		pickable: true
	};
	$: if (time >= endTime) resetTime();
	$: renderLayers(time);

	onMount(async () => {
		map = new maplibregl.Map({
			container: 'map', // container id
			style: mapStyle, // stylesheet location
			center: [114.157216, 22.362711], // starting position [lng, lat]
			zoom: 10.7 // starting zoom
		});
		map.addControl(new maplibregl.NavigationControl(), 'top-right');

		// Create the deck.gl overlay
		overlay = new MapboxOverlay({
			layers: [
				createTripsLayer(time),
				createPickupIconLayer(time),
				createDropoffIconLayer(time),
				createDepotIconLayer()
			],
			getTooltip: ({ object }) =>
				object &&
				`${object.stop_type} 
        request: ${object.request_id}
        time: ${minutesToTime(object.arrival_time)}
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
				createTripsLayer(time),
				createPickupIconLayer(time),
				createDropoffIconLayer(time),
				createDepotIconLayer()
			]
		});
	}

	function createTripsLayer(time) {
		return new TripsLayer({
			id: 'trips',
			data: filteredTrips,
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

	function createPickupIconLayer(time) {
		return new IconLayer({
			...iconLayerDefaultProps,
			id: 'pickup-icon',
			getIcon: getIconFn(pickupImg),
			// show for 30 unit time before arrival
			getFilterValue: (d) => d.arrival_time,
			filterRange: [time, time + 30],
			getFilterCategory: (d) => d.stop_type,
			filterCategories: ['pickup'],
			extensions: [new DataFilterExtension({ filterSize: 1, categorySize: 1 })]
		});
	}

	function createDropoffIconLayer(time) {
		return new IconLayer({
			...iconLayerDefaultProps,
			id: 'dropoff-icon',
			getIcon: getIconFn(dropoffImg),
			// show 30 unit time before arrival
			getFilterValue: (d) => d.arrival_time,
			filterRange: [time, time + 30],
			getFilterCategory: (d) => d.stop_type,
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
			// sum of number of elements in sample space of each category cannot exceed 128
			getFilterCategory: (d) => d.stop_type,
			filterCategories: ['depot'],
			// https://github.com/visgl/deck.gl/issues/9049#issuecomment-2253690167
			// Extension settings (categorySize in this case) are not designed to change on the fly.
			// Think of a layer + extension combination as a new type of layer - you can't change the same layer from Scatterplot to Text.
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
	<div class="map-bottom-bar">
		<div class="date">{date}</div>
		<div class="time">
			{minutesToTime(time)}
		</div>
		<div class="action-gp">
			{#if currentVehicle}
				<button
					on:click={() => {
						currentVehicle = null;
					}}>Show all</button
				>
			{/if}
			<select bind:value={currentVehicle}>
				<option value={null}>Show all</option>
				{#each trips as trip}
					<option value={trip}>Route of vehicle {trip.vehicle}</option>
				{/each}
			</select>
			{#if moving}
				<button on:click={() => stopVehicles(moveId)}>Stop</button>
			{:else}
				<button on:click={() => (moveId = runVehicles())}>Run</button>
			{/if}
		</div>
	</div>

	{#if !currentVehicle}
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
	{:else}
		<figure>
			<figcaption>Details of the vehicle {currentVehicle.vehicle}</figcaption>
			<div class="detail-wrapper">
				<figure class="summary">
					<figcaption>Summary</figcaption>
					<ul>
						{#each Object.entries(currentVehicle) as [key, value]}
							{#if !['route', 'path', 'original_path', 'time', 'original_time', 'vehicle'].includes(key)}
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
	{/if}
	<button class="request-btn" on:click={() => (showRequests = !showRequests)}>
		{#if showRequests}
			<span>Hide Handled Requests</span>
		{:else}
			<span>Show Handled Requests</span>
		{/if}
	</button>
	{#if showRequests}
		<figure class="summary">
			<figcaption>Details of Handled Requests</figcaption>
			<ul>
				{#each [...Array(summary.num_accepted_requests).keys()] as i}
					{@const pickup = stops[i]}
					{@const dropoff = stops[i + Number(summary.num_requests)]}
					<li>
						Request {i + 1}:
						<ul>
							<li>
								Pickup at {minutesToTime(pickup.arrival_time)}:
								<a href={getGoogleMapUrl(pickup.coordinates[1], pickup.coordinates[0])}
									>{pickup.coordinates[1]}, {pickup.coordinates[0]}</a
								>
							</li>
							<li>
								Dropoff at {minutesToTime(dropoff.arrival_time)}:
								<a href={getGoogleMapUrl(dropoff.coordinates[1], dropoff.coordinates[0])}
									>{dropoff.coordinates[1]}, {dropoff.coordinates[0]}</a
								>
							</li>
						</ul>
					</li>
				{/each}
			</ul>
		</figure>
	{/if}
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

	.map-bottom-bar {
		display: flex;
		justify-content: space-between;
		gap: 16px;
		font-size: 48px;
	}
	.map-bottom-bar > * {
		flex-grow: 1;
		flex-basis: 0;
	}
	.map-bottom-bar > div:nth-child(2) {
		text-align: center;
	}
	.action-gp {
		display: flex;
		justify-content: end;
		gap: 16px;
	}
	.action-gp > * {
		flex-grow: 1;
		flex-basis: 0;
		font-size: 24px;
		text-wrap: nowrap;
	}

	figure {
		margin: 16px 0;
		font-size: 24px;
	}
	figure.summary > ul {
		columns: 2;
	}

	.detail-wrapper {
		display: flex;
		gap: 32px;
	}

	button {
		font-size: 24px;
	}

	.request-btn {
		padding: 8px 16px;
		margin-bottom: 8px;
	}
</style>
