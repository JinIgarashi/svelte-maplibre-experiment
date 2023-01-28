<script lang="ts">
	import { onMount } from 'svelte';
	import { page } from '$app/stores';
	import maplibregl, {
		Map,
		NavigationControl,
		GeolocateControl,
		ScaleControl,
		AttributionControl,
		type LineLayerSpecification,
		type FillLayerSpecification,
		type MapGeoJSONFeature
	} from 'maplibre-gl';
	import * as pmtiles from 'pmtiles';

	import { map } from '$lib/stores';

	interface country {
		adm0_id: string;
		adm0_src: string;
		adm0_name: string;
		adm0_name1: string;
		iso_3: string;
	}
	let selectedCountry: MapGeoJSONFeature;

	let mapContainer: HTMLDivElement;

	onMount(async () => {
		const _map = new Map({
			container: mapContainer,
			style: `https://undp-data.github.io/style/aerialstyle.json`,
			center: [37.138, 0.414],
			zoom: 6,
			hash: true,
			attributionControl: false
		});
		_map.addControl(new NavigationControl({}), 'top-right');
		_map.addControl(
			new GeolocateControl({
				positionOptions: { enableHighAccuracy: true },
				trackUserLocation: true
			}),
			'top-right'
		);
		_map.addControl(new ScaleControl({ maxWidth: 80, unit: 'metric' }), 'bottom-left');
		_map.addControl(new AttributionControl({ compact: true }), 'bottom-right');

		_map.on('load', () => {
			let protocol = new pmtiles.Protocol();
			maplibregl.addProtocol('pmtiles', protocol.tile);

			_map.addSource('admin', {
				type: 'vector',
				url: `pmtiles://${$page.url.origin}/admin_0.pmtiles`,
				attribution:
					'"FieldMaps, United Nations, OCHA, geoBoundaries, U.S. Department of State, OpenStreetMap, UNFPA, Meta / Facebook, WorldPop"'
			});
		});

		map.update(() => _map);
	});

	$: selectedCountry, highlightCountry();
	const highlightCountry = () => {
		if (!selectedCountry) return;
		console.log(selectedCountry);
		const properties = selectedCountry.properties as country;
		const id = properties.adm0_src;

		const adminFillLayer: FillLayerSpecification = {
			id: 'admin-fill-shareburst',
			type: 'fill',
			source: 'admin',
			'source-layer': 'admin',
			filter: ['match', ['get', 'adm0_src'], id, false, true],
			paint: {
				'fill-color': 'rgba(211,211,211, 0.6)',
				'fill-outline-color': 'rgba(211,211,211, 0)'
			}
		};
		if ($map.getLayer(adminFillLayer.id)) {
			$map.removeLayer(adminFillLayer.id);
		}
		$map.addLayer(adminFillLayer);

		const adminLineLayer: LineLayerSpecification = {
			id: 'admin-line-shareburst',
			type: 'line',
			source: 'admin',
			'source-layer': 'admin',
			minzoom: 0,
			filter: ['match', ['get', 'adm0_src'], id, true, false],
			layout: {},
			paint: {
				'line-width': 5,
				'line-color': 'rgba(211,211,211, 0.6)',
				'line-blur': 2
			}
		};

		if ($map.getLayer(adminLineLayer.id)) {
			$map.removeLayer(adminLineLayer.id);
		}

		$map.addLayer(adminLineLayer);

		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		$map.fitBounds(selectedCountry.bbox);
	};

	const getCountryCodes = async () => {
		const res = await fetch(`/admin_0.geojson`);
		const json = await res.json();
		return json.features as MapGeoJSONFeature[];
	};

	const countryCodes = getCountryCodes();
</script>

<div class="map" id="map" bind:this={mapContainer}>
	<div class="panel p-2">
		{#await countryCodes then countries}
			<div class="select">
				<select bind:value={selectedCountry}>
					<option value={undefined}>Select a country to highlight</option>
					{#each countries as country}
						<option value={country}>{country.properties.adm0_name}</option>
					{/each}
				</select>
			</div>
		{/await}
	</div>
</div>

<style>
	@import 'maplibre-gl/dist/maplibre-gl.css';

	.map {
		position: absolute;
		top: 0;
		bottom: 0;
		width: 100%;
		z-index: 1;
	}

	.panel {
		position: absolute;
		top: 5px;
		left: 5px;
		background-color: white;
		width: fit-content;
		height: fit-content;
		z-index: 10;
	}
</style>
