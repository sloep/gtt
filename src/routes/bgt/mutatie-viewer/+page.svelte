<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import proj4 from 'proj4';
	import JSZip from 'jszip';
	import 'leaflet/dist/leaflet.css';

	// --- State Management ---
	let isDragging = $state(false);
	let isLoading = $state(false);
	let mutations: any[] = $state([]);
	let activeId: string | null = $state(null);
	let notifications: any[] = $state([]); // Voor waarschuwingen (zoals duplicaten)
	let sidebarWidth = $state(400); // Standaard breedte in pixels
	let isResizing = $state(false);

	// UI & Filters
	let activeTab = $state('lijst');
	let isFilterOpen = $state(false);
	let showAttributes = $state(true);
	let filters: Record<string, boolean> = $state({
		toevoeging: true,
		verwijderen: true,
		vorm: true,
		metadata: true,
		sleutel: true, // S: Sleutelwijziging
		ontdubbeling: true, // O: Ontdubbelen
		overig: true, // E, I, R: Relaties
		punt: true,
		lijn: true,
		vlak: true
	});

	// Derived State
	let stats = $derived({
		totaal: mutations.length,
		toevoeging: mutations.filter((m: any) => m.type === 'toevoeging').length,
		verwijderen: mutations.filter((m: any) => m.type === 'verwijderen').length,
		vorm: mutations.filter((m: any) => m.type === 'vorm').length,
		metadata: mutations.filter((m: any) => m.type === 'metadata').length,
		sleutel: mutations.filter((m: any) => m.type === 'sleutel').length,
		ontdubbeling: mutations.filter((m: any) => m.type === 'ontdubbeling').length,
		overig: mutations.filter((m: any) => m.type === 'overig').length,

		// Geometrie statistieken
		punt: mutations.filter((m: any) => m.geomType === 'punt').length,
		lijn: mutations.filter((m: any) => m.geomType === 'lijn').length,
		vlak: mutations.filter((m: any) => m.geomType === 'vlak').length,

		// Hoogteligging distributie (reduceert alle objecten naar een map van hoogtes)
		hoogteligging: mutations.reduce((acc: Record<string, number>, m: any) => {
			acc[m.hoogte] = (acc[m.hoogte] || 0) + 1;
			return acc;
		}, {})
	});

	// Update de gefilterde lijst met de nieuwe geometrie filters
	let filteredMutations = $derived(
		mutations.filter((m: any) => {
			const typeMatch = filters[m.type];
			const geomMatch = m.geomType === 'onbekend' || filters[m.geomType]; // Negeer objecten zonder geometrie in deze filter
			return typeMatch && geomMatch;
		})
	);

	// Imperatieve Kaart Variabelen (Buiten $state)
	let mapContainer: HTMLElement;
	let L: any;
	let map: any;
	let wasLayer: any;
	let wordtLayer: any;
	let leafletCache = new Map<string, any>();
	let seenSignatures = new Set<string>(); // Voor detectie van exacte duplicaten
	let dragCounter = 0;

	const STYLES: Record<string, any> = {
		toevoeging: { color: '#22c55e', weight: 3, fillColor: '#22c55e', fillOpacity: 0.5 },
		vervallen: {
			color: '#ef4444',
			weight: 3,
			fillColor: '#ef4444',
			fillOpacity: 0.3,
			dashArray: '4, 4'
		},
		was_contour: { color: '#ef4444', weight: 3, fillOpacity: 0, dashArray: '6, 6' },
		metadata: { color: '#f97316', weight: 3, fillColor: '#f97316', fillOpacity: 0.4 }
	};

	const ENTITEIT_TYPES: Record<string, string> = {
		BAK: 'Bak',
		BTD: 'begroeid terreindeel',
		BRD: 'Bord',
		BRT: 'buurt',
		FUG: 'functioneel gebied',
		GBI: 'gebouwinstallatie',
		INS: 'installatie',
		KST: 'Kast',
		KWD: 'kunstwerkdeel',
		MST: 'mast',
		OTD: 'onbegroeid terreindeel',
		OWT: 'ondersteunend waterdeel',
		OWG: 'ondersteunend wegdeel',
		OCO: 'ongeclassificeerd object',
		OPR: 'openbare ruimte',
		ORL: 'openbare ruimte label',
		OBD: 'overbruggingsdeel',
		OBW: 'overig bouwwerk',
		OSH: 'overige scheiding',
		PAL: 'Paal',
		PND: 'Pand',
		PBP: 'plaatsbepalingspunt',
		PUT: 'Put',
		SHD: 'scheiding',
		SNS: 'Sensor',
		SPR: 'Spoor',
		STD: 'stadsdeel',
		STM: 'straatmeubilair',
		TND: 'tunneldeel',
		VGO: 'vegetatieobject',
		WTD: 'waterdeel',
		WTI: 'waterinrichtingselement',
		WSP: 'waterschap',
		WGD: 'Wegdeel',
		WGI: 'Weginrichtingselement',
		WYK: 'Wijk'
	};

	onMount(async () => {
		if (!browser) return;

		L = (await import('leaflet')).default;
		proj4.defs(
			'EPSG:28992',
			'+proj=sterea +lat_0=52.15616055555555 +lon_0=5.38763888888889 +k=0.9999079 +x_0=155000 +y_0=463000 +ellps=bessel +towgs84=565.417,50.3319,465.552,-0.398957,0.343988,-1.8774,4.0725 +units=m +no_defs'
		);

		const mapConfig = { maxZoom: 24 };
		const osmMap = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			attribution: '© OpenStreetMap',
			maxNativeZoom: 19,
			...mapConfig
		});
		const luchtfotoMap = L.tileLayer(
			'https://service.pdok.nl/hwh/luchtfotorgb/wmts/v1_0/Actueel_orthoHR/EPSG:3857/{z}/{x}/{y}.jpeg',
			{ attribution: 'Luchtfoto © Kadaster', maxNativeZoom: 21, ...mapConfig }
		);
		const brtMap = L.tileLayer(
			'https://service.pdok.nl/brt/achtergrondkaart/wmts/v2_0/standaard/EPSG:3857/{z}/{x}/{y}.png',
			{ attribution: 'BRT © Kadaster', maxZoom: 16 }
		);
		const bgtMap = L.tileLayer(
			'https://service.pdok.nl/lv/bgt/wmts/v1_0/achtergrondvisualisatie/EPSG:3857/{z}/{x}/{y}.png',
			{ attribution: 'BGT © Kadaster', minZoom: 17, maxNativeZoom: 19, ...mapConfig }
		);

		const bgtBrtCombi = L.layerGroup([brtMap, bgtMap]);
		map = L.map(mapContainer, { layers: [bgtBrtCombi], maxZoom: 24 }).setView([52.09, 5.12], 8);

		wasLayer = L.featureGroup().addTo(map);
		wordtLayer = L.featureGroup().addTo(map);

		L.control
			.layers(
				{ 'PDOK BRT + BGT': bgtBrtCombi, 'PDOK Luchtfoto': luchtfotoMap, OpenStreetMap: osmMap },
				{ 'Situatie OUD': wasLayer, 'Situatie NIEUW': wordtLayer },
				{ position: 'topright' }
			)
			.addTo(map);
	});

	// Effecten & Meldingen
	$effect(() => {
		const currentMuts = filteredMutations;
		if (!wasLayer || !wordtLayer) return;
		wasLayer.clearLayers();
		wordtLayer.clearLayers();

		currentMuts.forEach((mut: any) => {
			const cache = leafletCache.get(mut.uniqueKey);
			if (cache) {
				cache.was.forEach((l: any) => wasLayer.addLayer(l));
				cache.wordt.forEach((l: any) => wordtLayer.addLayer(l));
			}
		});
	});

	function showNotification(msg: string, type: string = 'warning') {
		const id = Date.now();
		notifications.push({ id, msg, type });
		setTimeout(() => {
			notifications = notifications.filter((n: any) => n.id !== id);
		}, 6000);
	}

	// Geometrie & Acties
	const rdToWgs84 = (x: number, y: number) => proj4('EPSG:28992', 'EPSG:4326', [x, y]).reverse();
	const wgs84ToRd = (lat: number, lng: number) => proj4('EPSG:4326', 'EPSG:28992', [lng, lat]);

	function highlightMap(uniqueKey: string) {
		const cache = leafletCache.get(uniqueKey);
		if (!cache || (cache.was.length === 0 && cache.wordt.length === 0)) return;

		const group = L.featureGroup([...cache.was, ...cache.wordt]);
		map.fitBounds(group.getBounds(), { padding: [50, 50], maxZoom: 22 });

		[...cache.was, ...cache.wordt].forEach((layer: any) => {
			const origColor = layer.options.color;
			const origWeight = layer.options.originalWeight || 3;
			layer.setStyle({ color: '#facc15', weight: 6 });
			setTimeout(() => layer.setStyle({ color: origColor, weight: origWeight }), 800);
		});

		activeId = uniqueKey;
		document
			.getElementById(`tbl-${uniqueKey}`)
			?.scrollIntoView({ behavior: 'smooth', block: 'center' });
	}

	function openPdokViewer() {
		if (!map) return;
		const center = map.getCenter();
		const pdokZoom = map.getZoom() - 4.5856;
		const [rdX, rdY] = wgs84ToRd(center.lat, center.lng);
		window.open(
			`https://app.pdok.nl/viewer/#x=${rdX.toFixed(2)}&y=${rdY.toFixed(2)}&z=${pdokZoom.toFixed(4)}&background=Luchtfoto&layers=2891cc29-0a79-46d1-8649-287046d621c7;standaardvisualisatie;_;0.5,2891cc29-0a79-46d1-8649-287046d621c7;icoonvisualisatie;_;1`,
			'_blank'
		);
	}

	// XML Parsers
	const getSafeText = (node: Element, name: string) =>
		Array.from(node.getElementsByTagName('*')).find(
			(el: Element) => el.localName === name || el.tagName.endsWith(`:${name}`)
		)?.textContent || null;

	const getGeomString = (node: Element | null) =>
		!node
			? ''
			: Array.from(node.getElementsByTagName('*'))
					.filter((n: Element) => n.localName === 'posList' || n.localName === 'pos')
					.map((n: Element) => (n.textContent || '').trim().replace(/\s+/g, ' '))
					.join('|');

	const isTerminated = (node: Element) => {
		const n = Array.from(node.children).find(
			(c: Element) => c.localName === 'terminationDate' || c.tagName.endsWith(':terminationDate')
		);
		return n && !n.getAttribute('xsi:nil') && (n.textContent || '').trim() !== '';
	};

	const extractAttributes = (node: Element | null, prefix = ''): Record<string, string> => {
		if (!node) return {};
		const attr: Record<string, string> = {};

		for (const c of Array.from(node.children)) {
			const name = c.localName || c.tagName.replace(/^[^:]+:/, '');

			// Sla geometrie en identificatie (overhead) over, dit voorkomt gigantische tabellen
			if (
				['geometrie', 'identificatie', 'parameters', 'poslist', 'pos', 'voorkomen'].includes(
					name.toLowerCase()
				)
			) {
				continue;
			}

			const key = prefix ? `${prefix}.${name}` : name;

			if (c.children.length === 0) {
				attr[key] = (c.textContent || '').trim();
			} else {
				// Recursie: Duik dieper in de XML node en voeg ze toe aan de parent
				Object.assign(attr, extractAttributes(c, key));
			}
		}
		return attr;
	};

	function createGeometryLayers(node: Element | null, styleKey: string, uniqueKey: string) {
		if (!node || !L) return [];
		const styling = { ...STYLES[styleKey], originalWeight: STYLES[styleKey].weight };
		const createdLayers: any[] = [];

		Array.from(node.getElementsByTagName('*'))
			.filter((n: Element) => n.localName === 'posList' || n.localName === 'pos')
			.forEach((posNode: Element) => {
				const coords = (posNode.textContent || '').trim().split(/\s+/).map(Number);
				const latLngs: any[] = [];
				for (let i = 0; i < coords.length; i += 2)
					if (!isNaN(coords[i]) && !isNaN(coords[i + 1]))
						latLngs.push(rdToWgs84(coords[i], coords[i + 1]));

				let layer: any = null;
				if (latLngs.length === 1)
					layer = L.circleMarker(latLngs[0], {
						radius: 6,
						...styling,
						fillOpacity: styleKey === 'was_contour' ? 0 : 0.8
					});
				else if (latLngs.length > 1) {
					const isClosed =
						Math.abs(latLngs[0][0] - latLngs[latLngs.length - 1][0]) < 0.0001 &&
						Math.abs(latLngs[0][1] - latLngs[latLngs.length - 1][1]) < 0.0001;
					layer =
						isClosed && latLngs.length > 3
							? L.polygon(latLngs, styling)
							: L.polyline(latLngs, styling);
				}
				if (layer) {
					layer.on('click', () => highlightMap(uniqueKey));
					createdLayers.push(layer);
				}
			});
		return createdLayers;
	}

	const isSemanticChange = (valWas: string, valWordt: string): boolean => {
		if (valWas === valWordt) return false;

		const numWas = Number(valWas);
		const numWordt = Number(valWordt);
		if (!isNaN(numWas) && !isNaN(numWordt) && numWas === numWordt) return false;

		if (
			(valWas.includes('-') || valWordt.includes('-')) &&
			!isNaN(Date.parse(valWas)) &&
			!isNaN(Date.parse(valWordt))
		) {
			if (Date.parse(valWas) === Date.parse(valWordt)) return false;
		}

		if (valWas.toLowerCase().trim() === valWordt.toLowerCase().trim()) return false;

		return true; // Ze zijn semantisch écht verschillend
	};

	const getGeometryType = (node: Element | null): string => {
		if (!node) return 'onbekend';
		const elements = Array.from(node.getElementsByTagName('*'));

		// Controleer op formele GML entiteiten
		if (
			elements.some((n: Element) =>
				['Polygon', 'Surface', 'MultiSurface'].includes(n.localName || '')
			)
		)
			return 'vlak';
		if (elements.some((n: Element) => ['LineString', 'Curve'].includes(n.localName || '')))
			return 'lijn';
		if (elements.some((n: Element) => n.localName === 'Point')) return 'punt';

		// Fallback gebaseerd op het aantal coördinaten
		const posNode = elements.find(
			(n: Element) => n.localName === 'posList' || n.localName === 'pos'
		);
		if (!posNode) return 'onbekend';
		if (posNode.localName === 'pos') return 'punt';
		const coords = (posNode.textContent || '').trim().split(/\s+/);
		return coords.length < 6 ? 'lijn' : 'vlak';
	};

	function parseXMLData(xmlString: string, parseStats: any) {
		const parser = new DOMParser();
		const xmlDoc = parser.parseFromString(xmlString, 'text/xml');
		const objects = Array.from(xmlDoc.getElementsByTagName('*')).filter(
			(el) => el.localName === 'object' || el.tagName.endsWith(':object')
		);

		const groups: Record<string, any[]> = {};
		objects.forEach((obj) => {
			const lokaalID = getSafeText(obj, 'lokaalID');
			const soortAttr = Array.from(obj.attributes).find((a) => a.name.endsWith('verwerkingssoort'));
			const entiteitAttr = Array.from(obj.attributes).find((a) => a.name.endsWith('entiteittype'));

			if (lokaalID && soortAttr) {
				groups[lokaalID] = groups[lokaalID] || [];
				groups[lokaalID].push({
					node: obj,
					soort: soortAttr.value,
					entiteit: entiteitAttr ? entiteitAttr.value : '?'
				});
			}
		});

		const parsedMutations: any[] = [];

		for (const [id, items] of Object.entries(groups)) {
			items.sort((a: any, b: any) =>
				(getSafeText(a.node, 'tijdstipRegistratie') || '').localeCompare(
					getSafeText(b.node, 'tijdstipRegistratie') || ''
				)
			);

			const nodeWas = items[0].node;
			const nodeWordt = items[items.length - 1].node;
			const soortWordt = items[items.length - 1].soort; // Bevat de StUF letter (T,V,W,etc)
			const registratieTijd = getSafeText(nodeWordt, 'tijdstipRegistratie') || 'onbekend';
			const entiteitCode = items[items.length - 1].entiteit;
			const entiteitNaam = ENTITEIT_TYPES[entiteitCode] || 'Onbekend objecttype';

			// Detectie van exacte duplicaten (Zelfde ID + Zelfde Registratietijd)
			const signature = `${id}-${registratieTijd}`;
			if (seenSignatures.has(signature)) {
				parseStats.duplicatesSkipped++;
				continue;
			}
			seenSignatures.add(signature);

			const isDel = isTerminated(nodeWordt) || soortWordt === 'V';
			const uniqueKey = `${id}-${Math.random().toString(36).substring(2, 11)}`;

			let type = '',
				badge = '',
				borderColor = '';
			const layersWas: any[] = [],
				layersWordt: any[] = [];

			if (soortWordt === 'T' || (items.length === 1 && soortWordt === 'T')) {
				type = 'toevoeging';
				badge = 'Toevoegen (T)';
				borderColor = 'border-l-green-500';
				layersWordt.push(...createGeometryLayers(nodeWordt, 'toevoeging', uniqueKey));
			} else if (isDel) {
				type = 'verwijderen';
				badge = 'Verwijderen (V)';
				borderColor = 'border-l-red-500';
				layersWas.push(...createGeometryLayers(nodeWas, 'vervallen', uniqueKey));
			} else if (soortWordt === 'W') {
				// logica om 'W' te splitsen in Vorm vs Metadata
				if (getGeomString(nodeWas) === getGeomString(nodeWordt)) {
					type = 'metadata';
					badge = 'Wijziging: Metadata (W)';
					borderColor = 'border-l-orange-400';
					layersWordt.push(...createGeometryLayers(nodeWordt, 'metadata', uniqueKey));
				} else {
					type = 'vorm';
					badge = 'Wijziging: Vorm (W)';
					borderColor = 'border-l-yellow-400';
					layersWas.push(...createGeometryLayers(nodeWas, 'was_contour', uniqueKey));
					layersWordt.push(...createGeometryLayers(nodeWordt, 'toevoeging', uniqueKey));
				}
			} else if (soortWordt === 'S') {
				type = 'sleutel';
				badge = 'Sleutelwijziging (S)';
				borderColor = 'border-l-blue-500';
				layersWordt.push(...createGeometryLayers(nodeWordt, 'metadata', uniqueKey));
			} else if (soortWordt === 'O') {
				type = 'ontdubbeling';
				badge = 'Ontdubbeling (O)';
				borderColor = 'border-l-purple-500';
				layersWordt.push(...createGeometryLayers(nodeWordt, 'metadata', uniqueKey));
			} else {
				type = 'overig';
				badge = `Relatie/Overig (${soortWordt || '?'})`;
				borderColor = 'border-l-gray-400';
				layersWordt.push(...createGeometryLayers(nodeWordt, 'metadata', uniqueKey));
			}

			// Veilig opslaan in cache d.m.v. uniqueKey
			leafletCache.set(uniqueKey, { was: layersWas, wordt: layersWordt });

			const attrWordt: Record<string, string> = extractAttributes(nodeWordt);
			const attrWas: Record<string, string> =
				type === 'toevoeging' ? {} : extractAttributes(nodeWas);
			const keys = new Set([...Object.keys(attrWas), ...Object.keys(attrWordt)]);
			const diffRows: any[] = [];

			keys.forEach((key) => {
				const vW = attrWas[key] || '-';
				const vT = attrWordt[key] || '-';

				const changed = isSemanticChange(vW, vT);

				diffRows.push({ key, vW, vT, changed });
			});

			// zet alle gemuteerde gewijzigde regels bovenaan
			diffRows.sort((a, b) => {
				if (a.changed === b.changed) return a.key.localeCompare(b.key);
				return a.changed ? -1 : 1;
			});

			const targetNode = isDel ? nodeWas : nodeWordt;
			const targetAttr = isDel ? attrWas : attrWordt;

			const geomType = getGeometryType(targetNode);

			// relatieve hoogteligging (standaard 0 voor maaiveld)
			const hKey = Object.keys(targetAttr).find((k) =>
				k.toLowerCase().includes('relatievehoogteligging')
			);
			const hoogte =
				hKey && !isNaN(parseInt(targetAttr[hKey], 10)) ? parseInt(targetAttr[hKey], 10) : 0;

			parsedMutations.push({
				id,
				uniqueKey,
				type,
				badge,
				borderColor,
				diffRows,
				geomType,
				hoogte,
				entiteitCode,
				entiteitNaam
			});
		}
		return parsedMutations;
	}

	// File & UI Handlers
	function resetUI() {
		isLoading = true;
		mutations = [];
		activeId = null;
		leafletCache.clear();
		seenSignatures.clear();
		if (wasLayer) wasLayer.clearLayers();
		if (wordtLayer) wordtLayer.clearLayers();
	}

	function finalizeUI(parseStats: any) {
		if (mutations.length > 0) {
			const allLayers = Array.from(leafletCache.values()).flatMap((c: any) => [
				...c.was,
				...c.wordt
			]);
			if (allLayers.length > 0) {
				const bounds = L.featureGroup(allLayers).getBounds();
				if (bounds.isValid()) map.fitBounds(bounds, { padding: [50, 50], maxZoom: 22 });
			}
		}

		if (parseStats.duplicatesSkipped > 0) {
			showNotification(
				`Er zijn ${parseStats.duplicatesSkipped} exact dubbele mutaties overgeslagen.`
			);
		}
		isLoading = false;
	}

	async function processFiles(files: any) {
		if (!files || files.length === 0) return;
		resetUI();
		let newMutations: any[] = [];
		let parseStats = { duplicatesSkipped: 0 };

		for (let file of files) {
			if (file.name.endsWith('.zip')) {
				const zip = await JSZip.loadAsync(file);
				for (const filename of Object.keys(zip.files)) {
					if (filename.endsWith('.xml')) {
						newMutations.push(
							...parseXMLData(await zip.files[filename].async('string'), parseStats)
						);
						await new Promise((r) => setTimeout(r, 10));
					}
				}
			} else if (file.name.endsWith('.xml')) {
				newMutations.push(...parseXMLData(await file.text(), parseStats));
			}
		}
		mutations = newMutations;
		finalizeUI(parseStats);
	}

	async function loadSample() {
		const url = '/samples/mtbVerticaal.xml';

		resetUI();
		let parseStats = { duplicatesSkipped: 0 };

		try {
			const response = await fetch(url);
			if (!response.ok) throw new Error(`HTTP fout! Status: ${response.status}`);
			const xmlStr = await response.text();

			mutations = parseXMLData(xmlStr, parseStats);
			finalizeUI(parseStats);
		} catch (err) {
			const error = err as Error;
			showNotification(`Fout bij het inladen van het BGT-bestand: ${error.message}`, 'error');
			isLoading = false;
		}
	}

	function handleDrag(e: DragEvent, entering: boolean) {
		e.preventDefault();
		dragCounter += entering ? 1 : -1;
		isDragging = dragCounter > 0;
	}
	function handleDrop(e: DragEvent) {
		e.preventDefault();
		dragCounter = 0;
		isDragging = false;
		if (e.dataTransfer) processFiles(e.dataTransfer.files);
	}

	// Sidebar Resize Logica
	function startResize(e: MouseEvent) {
		isResizing = true;
		document.body.style.userSelect = 'none'; // Voorkom irritante tekstselectie tijdens slepen
		document.body.style.cursor = 'ew-resize';
	}

	function handleResize(e: MouseEvent) {
		if (!isResizing) return;
		// Begrens de breedte tussen 250px en 800px zodat de UI niet breekt
		sidebarWidth = Math.max(250, Math.min(800, e.clientX));
	}

	function stopResize() {
		if (isResizing) {
			isResizing = false;
			document.body.style.userSelect = '';
			document.body.style.cursor = '';
		}
	}
</script>

<svelte:window
	ondragenter={(e) => handleDrag(e, true)}
	ondragleave={(e) => handleDrag(e, false)}
	ondragover={(e) => e.preventDefault()}
	ondrop={handleDrop}
	onmousemove={handleResize}
	onmouseup={stopResize}
/>

<div class="fixed top-4 right-4 z-500 flex flex-col gap-2 pointer-events-none">
	{#each notifications as note (note.id)}
		<div
			class="pointer-events-auto bg-white border-l-4 shadow-lg rounded px-4 py-3 min-w-75 flex items-start gap-3 transform transition-all
            {note.type === 'error' ? 'border-red-500' : 'border-yellow-500'}"
		>
			{#if note.type === 'error'}
				<svg
					class="w-5 h-5 text-red-500 mt-0.5"
					fill="none"
					stroke="currentColor"
					viewBox="0 0 24 24"
					><path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
					></path></svg
				>
			{:else}
				<svg
					class="w-5 h-5 text-yellow-500 mt-0.5"
					fill="none"
					stroke="currentColor"
					viewBox="0 0 24 24"
					><path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
					></path></svg
				>
			{/if}
			<div>
				<h4 class="font-bold text-gray-800 text-sm">
					{note.type === 'error' ? 'Foutmelding' : 'Let op'}
				</h4>
				<p class="text-sm text-gray-600">{note.msg}</p>
			</div>
		</div>
	{/each}
</div>

<div class="bg-gray-100 text-gray-800 flex flex-col h-screen overflow-hidden relative font-sans">
	{#if isDragging}
		<div
			class="fixed inset-0 z-100 bg-blue-600/80 backdrop-blur-sm flex flex-col items-center justify-center text-white pointer-events-none transition-opacity"
		>
			<h2 class="text-4xl font-bold drop-shadow-md animate-bounce">Laat bestand(en) hier los</h2>
		</div>
	{/if}

	{#if isLoading}
		<div
			class="fixed inset-0 z-60 bg-white/80 backdrop-blur-sm flex flex-col items-center justify-center"
		>
			<div class="animate-spin rounded-full h-16 w-16 border-b-4 border-blue-600 mb-4"></div>
			<p class="text-lg font-semibold text-gray-700">Verwerken...</p>
		</div>
	{/if}

	<header class="bg-white shadow z-10 py-3 px-6 flex justify-between items-center shrink-0">
		<div>
			<h1 class="text-xl font-bold text-gray-900">BGT Mutatie Viewer</h1>
			<p class="text-xs text-gray-500">StUF Di01 mutatie- en abonnementsbestanden</p>
		</div>
		<div class="flex gap-2">
			<button
				onclick={openPdokViewer}
				class="bg-indigo-50 border border-indigo-200 hover:bg-indigo-100 text-indigo-700 px-4 py-2 rounded text-sm font-medium"
				>PDOK Viewer</button
			>
			<button
				onclick={loadSample}
				class="bg-gray-200 hover:bg-gray-300 text-gray-800 px-4 py-2 rounded transition-colors font-medium text-sm shadow-sm"
			>
				Laad voorbeeld
			</button>
			<label
				class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded cursor-pointer text-sm font-medium shadow-sm"
			>
				Bestanden Openen... <input
					type="file"
					onchange={(e) => processFiles((e.target as HTMLInputElement)?.files)}
					class="hidden"
					accept=".xml,.zip"
					multiple
				/>
			</label>
		</div>
	</header>

	<main class="flex-1 flex overflow-hidden">
		<aside
			class="bg-white flex flex-col shadow-inner z-20 shrink-0"
			style="width: {sidebarWidth}px;"
		>
			<div class="flex border-b bg-gray-50 shrink-0 text-sm font-medium">
				<button
					class="flex-1 py-3 text-center border-b-2 transition-colors {activeTab === 'lijst'
						? 'border-blue-600 text-blue-700 bg-white'
						: 'border-transparent text-gray-500 hover:text-gray-700 hover:bg-gray-100'}"
					onclick={() => (activeTab = 'lijst')}>Lijstweergave</button
				>
				<button
					class="flex-1 py-3 text-center border-b-2 transition-colors {activeTab === 'stats'
						? 'border-blue-600 text-blue-700 bg-white'
						: 'border-transparent text-gray-500 hover:text-gray-700 hover:bg-gray-100'}"
					onclick={() => (activeTab = 'stats')}>Statistieken</button
				>
			</div>

			{#if activeTab === 'lijst'}
				<div class="p-3 border-b bg-white shrink-0 space-y-2 relative">
					<div class="flex gap-2">
						<button
							class="flex-1 flex justify-between items-center bg-gray-100 hover:bg-gray-200 px-3 py-1.5 rounded text-sm text-gray-700 transition"
							onclick={() => (isFilterOpen = !isFilterOpen)}
						>
							<span>Filter Type</span>
							<svg
								class="w-4 h-4 transform transition-transform {isFilterOpen ? 'rotate-180' : ''}"
								fill="none"
								stroke="currentColor"
								viewBox="0 0 24 24"
								><path
									stroke-linecap="round"
									stroke-linejoin="round"
									stroke-width="2"
									d="M19 9l-7 7-7-7"
								></path></svg
							>
						</button>
						<button
							class="flex-1 text-center border px-3 py-1.5 rounded text-sm transition {showAttributes
								? 'bg-blue-50 border-blue-200 text-blue-700'
								: 'bg-gray-50 text-gray-500'}"
							onclick={() => (showAttributes = !showAttributes)}
						>
							{showAttributes ? 'Verberg Tabellen' : 'Toon Tabellen'}
						</button>
					</div>

					{#if isFilterOpen}
						<div
							class="absolute top-full left-3 right-3 bg-white border rounded shadow-lg p-3 z-50 flex flex-col gap-2 text-sm mt-1"
						>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.toevoeging} class="text-green-600" /> Toevoegen
									(T)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.toevoeging}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.verwijderen} class="text-red-600" /> Verwijderen
									/ Vervallen (V)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.verwijderen}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.vorm} class="text-yellow-500" /> Wijzigen:
									Vorm (W)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.vorm}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.metadata} class="text-orange-500" /> Wijzigen:
									Metadata (W)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.metadata}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.sleutel} class="text-blue-500" /> Sleutelwijziging
									(S)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.sleutel}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input
										type="checkbox"
										bind:checked={filters.ontdubbeling}
										class="text-purple-500"
									/> Ontdubbeling (O)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.ontdubbeling}</span>
							</label>
							<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
								<span class="flex items-center gap-2">
									<input type="checkbox" bind:checked={filters.overig} class="text-gray-500" /> Relatie
									/ Overig (E, I, R)
								</span>
								<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.overig}</span>
							</label>
							{#if stats.punt > 0 || stats.lijn > 0 || stats.vlak > 0}
								<div
									class="border-t border-gray-200 mt-2 pt-2 pb-1 text-xs font-bold text-gray-500 uppercase tracking-wide"
								>
									Geometrie Vorm
								</div>
								{#if stats.vlak > 0}
									<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
										<span class="flex items-center gap-2"
											><input type="checkbox" bind:checked={filters.vlak} class="text-blue-600" /> Vlakken</span
										>
										<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.vlak}</span>
									</label>
								{/if}
								{#if stats.lijn > 0}
									<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
										<span class="flex items-center gap-2"
											><input type="checkbox" bind:checked={filters.lijn} class="text-blue-600" /> Lijnen</span
										>
										<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.lijn}</span>
									</label>
								{/if}
								{#if stats.punt > 0}
									<label class="flex justify-between cursor-pointer hover:bg-gray-50 p-1 rounded">
										<span class="flex items-center gap-2"
											><input type="checkbox" bind:checked={filters.punt} class="text-blue-600" /> Punten</span
										>
										<span class="bg-gray-100 px-2 rounded-full text-xs">{stats.punt}</span>
									</label>
								{/if}
							{/if}
						</div>
					{/if}
				</div>

				<div class="flex-1 overflow-y-auto p-4 space-y-4 pb-20 bg-gray-50">
					{#if filteredMutations.length === 0}
						<p class="text-center text-gray-400 mt-10 text-sm">
							Geen objecten zichtbaar. Open of sleep een geldig bestand om in te laden.
						</p>
					{:else}
						{#each filteredMutations as mut (mut.uniqueKey)}
							<div
								id="tbl-{mut.uniqueKey}"
								class="bg-white border rounded shadow-sm overflow-hidden border-l-4 {mut.borderColor} cursor-pointer hover:shadow-md transition-all {activeId ===
								mut.uniqueKey
									? 'ring-2 ring-blue-400 scale-[1.02]'
									: ''}"
								role="button"
								tabindex="0"
								onclick={() => highlightMap(mut.uniqueKey)}
								onkeydown={(e) => {
									if (e.key === 'Enter' || e.key === ' ') highlightMap(mut.uniqueKey);
								}}
							>
								<div
									class="bg-gray-50 px-3 py-2 text-xs text-gray-500 border-b flex justify-between items-center gap-2"
								>
									<span class="truncate font-mono" title={mut.id}>
										{mut.id.split('.')[1] || mut.id}
									</span>
									<div class="flex gap-1.5 ml-auto">
										<span
											class="font-bold whitespace-nowrap bg-indigo-50 border border-indigo-200 text-indigo-700 px-1.5 py-0.5 rounded text-[10px] cursor-help"
											title={mut.entiteitNaam}
										>
											{mut.entiteitCode}
										</span>
										<span
											class="font-bold whitespace-nowrap bg-white border px-1.5 py-0.5 rounded text-[10px]"
										>
											{mut.badge}
										</span>
									</div>
								</div>
								{#if showAttributes}
									{#if mut.diffRows.length > 0}
										<table class="w-full text-left table-fixed text-[11px]">
											<tbody class="divide-y divide-gray-100">
												{#each mut.diffRows as row}
													<tr class="{row.changed ? 'bg-yellow-50' : ''} hover:bg-gray-100">
														<td class="py-1 px-2 text-gray-500 w-1/3 truncate" title={row.key}
															>{row.key}</td
														>
														<td
															class="py-1 px-2 {row.changed
																? 'font-bold text-gray-900'
																: 'text-gray-600'} w-1/3 truncate">{row.vW}</td
														>
														<td
															class="py-1 px-2 {row.changed
																? 'font-bold text-gray-900'
																: 'text-gray-600'} w-1/3 truncate">{row.vT}</td
														>
													</tr>
												{/each}
											</tbody>
										</table>
									{:else}
										<div class="p-2 text-center text-[10px] text-gray-400 italic">
											Geen attribuutwijzigingen.
										</div>
									{/if}
								{/if}
							</div>
						{/each}
					{/if}
				</div>
			{/if}

			{#if activeTab === 'stats'}
				<div class="flex-1 overflow-y-auto p-6 bg-white">
					<h2 class="text-2xl font-bold mb-6 text-gray-800 border-b pb-2">Overzicht</h2>

					<div class="bg-blue-50 border border-blue-100 p-4 rounded-lg text-center mb-6">
						<div class="text-4xl font-extrabold text-blue-600">{stats.totaal}</div>
						<div class="text-sm font-medium text-blue-800 uppercase tracking-wide mt-1">
							Totale Mutaties
						</div>
					</div>

					<div class="space-y-4">
						{#if stats.toevoeging > 0}
							<div
								class="flex justify-between items-center p-3 bg-green-50 rounded border border-green-100"
							>
								<span class="font-medium text-green-800">Toevoegingen (T)</span>
								<span class="bg-green-200 text-green-900 px-3 py-1 rounded-full font-bold"
									>{stats.toevoeging}</span
								>
							</div>
						{/if}

						{#if stats.verwijderen > 0}
							<div
								class="flex justify-between items-center p-3 bg-red-50 rounded border border-red-100"
							>
								<span class="font-medium text-red-800">Verwijderen / Vervallen (V)</span>
								<span class="bg-red-200 text-red-900 px-3 py-1 rounded-full font-bold"
									>{stats.verwijderen}</span
								>
							</div>
						{/if}

						{#if stats.vorm > 0}
							<div
								class="flex justify-between items-center p-3 bg-yellow-50 rounded border border-yellow-100"
							>
								<span class="font-medium text-yellow-800">Vormwijzigingen (W)</span>
								<span class="bg-yellow-200 text-yellow-900 px-3 py-1 rounded-full font-bold"
									>{stats.vorm}</span
								>
							</div>
						{/if}

						{#if stats.metadata > 0}
							<div
								class="flex justify-between items-center p-3 bg-orange-50 rounded border border-orange-100"
							>
								<span class="font-medium text-orange-800">Enkel Metadata (W)</span>
								<span class="bg-orange-200 text-orange-900 px-3 py-1 rounded-full font-bold"
									>{stats.metadata}</span
								>
							</div>
						{/if}

						{#if stats.sleutel > 0}
							<div
								class="flex justify-between items-center p-3 bg-blue-50 rounded border border-blue-100"
							>
								<span class="font-medium text-blue-800">Sleutelwijziging (S)</span>
								<span class="bg-blue-200 text-blue-900 px-3 py-1 rounded-full font-bold"
									>{stats.sleutel}</span
								>
							</div>
						{/if}

						{#if stats.ontdubbeling > 0}
							<div
								class="flex justify-between items-center p-3 bg-purple-50 rounded border border-purple-100"
							>
								<span class="font-medium text-purple-800">Ontdubbeling (O)</span>
								<span class="bg-purple-200 text-purple-900 px-3 py-1 rounded-full font-bold"
									>{stats.ontdubbeling}</span
								>
							</div>
						{/if}

						{#if stats.overig > 0}
							<div
								class="flex justify-between items-center p-3 bg-gray-50 rounded border border-gray-200"
							>
								<span class="font-medium text-gray-800">Relatie / Overig (E, I, R)</span>
								<span class="bg-gray-200 text-gray-900 px-3 py-1 rounded-full font-bold"
									>{stats.overig}</span
								>
							</div>
						{/if}

						{#if Object.keys(stats.hoogteligging).some((niveau) => niveau !== '0')}
							<div class="mt-8 pt-4 border-t border-gray-200">
								<h3 class="text-sm font-bold text-gray-700 mb-3 uppercase tracking-wide">
									Relatieve Hoogteligging
								</h3>
								<div class="space-y-2">
									{#each Object.entries(stats.hoogteligging).sort((a, b) => Number(b[0]) - Number(a[0])) as [niveau, aantal]}
										<div class="flex items-center gap-3">
											<div
												class="w-8 text-right font-mono text-sm font-bold {Number(niveau) === 0
													? 'text-gray-400'
													: Number(niveau) > 0
														? 'text-blue-600'
														: 'text-orange-600'}"
												title="Niveau {niveau}"
											>
												{Number(niveau) > 0 ? '+' : ''}{niveau}
											</div>
											<div class="flex-1 bg-gray-100 rounded-full h-3 overflow-hidden flex">
												<div
													class="h-full bg-slate-400 rounded-full transition-all duration-500"
													style="width: {Math.max(1, (aantal / stats.totaal) * 100)}%"
												></div>
											</div>
											<div class="w-8 text-right text-xs font-semibold text-gray-600">{aantal}</div>
										</div>
									{/each}
								</div>
								<p class="text-[10px] text-gray-400 mt-2 text-center">Niveau 0 = Maaiveld</p>
							</div>
						{/if}

						{#if stats.totaal === 0}
							<div class="text-center text-gray-400 italic text-sm mt-4">
								Geen mutaties om weer te geven.
							</div>
						{/if}
					</div>
				</div>
			{/if}
		</aside>

		<button
			class="w-1.5 bg-gray-200 hover:bg-blue-400 cursor-ew-resize z-30 shrink-0 border-x border-gray-300 transition-colors p-0 {isResizing
				? 'bg-blue-500'
				: ''}"
			onmousedown={startResize}
			aria-label="Pas menubreedte aan"
			title="Pas menubreedte aan"
		></button>

		<section class="flex-1 h-full relative z-10">
			<div bind:this={mapContainer} class="w-full h-full bg-gray-200"></div>

			<div
				class="absolute bottom-6 left-6 bg-white/95 backdrop-blur p-3 rounded shadow-lg border border-gray-200 text-xs pointer-events-none z-400"
			>
				<h3 class="font-bold mb-2 text-gray-800 border-b pb-1">Legenda Kaart</h3>
				<div class="flex items-center gap-2">
					<span class="w-4 h-4 bg-red-500 border border-red-700 opacity-40"></span> Was / Vervallen
				</div>
				<div class="flex items-center gap-2 mt-1">
					<span class="w-4 h-4 bg-green-500 border border-green-700 opacity-60"></span> Wordt / Toevoeging
				</div>
				<div class="flex items-center gap-2 mt-1">
					<span class="w-4 h-4 bg-transparent border-2 border-dashed border-red-500 opacity-60"
					></span> Oude vorm (bij Vormwijziging)
				</div>
				<div class="flex items-center gap-2 mt-1">
					<span class="w-4 h-4 bg-orange-400 border border-orange-600 opacity-60"></span> Metadata-wijziging
				</div>
			</div>
		</section>
	</main>
</div>
