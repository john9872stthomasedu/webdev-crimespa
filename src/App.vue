<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);
let location_input = ref('');
let is_programmatic_move = ref(false);
let user_has_interacted = ref(false);

// New incident form data
let incident_form = reactive({
    date: '',
    time: '',
    address: '',
    latitude: '',
    longitude: '',
    crime_code: '',
    description: '',
    neighborhood: ''
});

let incident_form_errors = ref({});
let incident_form_submitting = ref(false);
let incident_form_success = ref(false);
let incident_form_error_message = ref('');
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
        ]
    }
);

let codes_by_code = ref({});
let neighborhoods_by_id = ref({});
let crimes = ref([]);
let neighborhood_crime_counts = ref({});

// --- FILTER STATE & HELPERS (add below your other reactive state like codes_by_code/neighborhoods_by_id) ---
let filters = reactive({
    incident_types: {},   // { "Theft": true, "Narcotics": true, ... }
    neighborhoods: {},    // { "Downtown": true, ... }
    start_date: '',       // 'YYYY-MM-DD'
    end_date: '',
    max_incidents: 1000
});

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
    });

    // Track when user manually pans/zooms the map
    map.leaflet.on('movestart', () => {
        if (!is_programmatic_move.value) {
            user_has_interacted.value = true;
        }
    });

    // Update location input only after user manually pans/zooms
    map.leaflet.on('moveend', () => {
        if (!is_programmatic_move.value && user_has_interacted.value) {
            updateLocationInputFromMap();
        }
    });
});



// ---- FETCH HELPERS (avoid `response.json()` crashing on empty/invalid JSON) ----
async function fetchJsonSafe(url, options) {
    const resp = await fetch(url, options);
    const raw = await resp.text().catch(() => '');
    if (!resp.ok) {
        throw new Error(`HTTP ${resp.status} ${resp.statusText} ${raw}`.trim());
    }
    if (!raw) return { resp, data: null, raw: '' };
    try {
        return { resp, data: JSON.parse(raw), raw };
    } catch (e) {
        console.error('Invalid JSON from', url, raw);
        throw e;
    }
}

// FUNCTIONS
// Function called once user has entered REST API URL
async function initializeCrimes() {
    if (!crime_url || !crime_url.value) {
        console.warn('initializeCrimes: crime_url not set');
        return;
    }

    const base = String(crime_url.value).replace(/\/$/, '');
    console.info('initializeCrimes: using base URL', base);

    // 1) fetch lookups (best-effort)
    try {
        const [{ data: codesArr }, { data: nbhArr }] = await Promise.all([
            fetchJsonSafe(`${base}/codes`),
            fetchJsonSafe(`${base}/neighborhoods`)
        ]);


        const codeMap = {};
        if (Array.isArray(codesArr)) {
            codesArr.forEach(c => {
                if (c && c.code != null) codeMap[String(c.code)] = c.incident_type ?? c.incident ?? c.description ?? '';
            });
        }
        codes_by_code.value = codeMap;

        const nbhMap = {};
        if (Array.isArray(nbhArr)) {
            nbhArr.forEach(n => {
                const id = n.neighborhood_number ?? n.id ?? n.neighborhood_id;
                const name = n.neighborhood_name ?? n.name ?? '';
                if (id != null) nbhMap[String(id)] = name;
            });
        }
        neighborhoods_by_id.value = nbhMap;
        populateFilterLists();
    } catch (err) {
        console.warn('initializeCrimes: lookup fetch failed (continuing):', err);
        // keep empty maps if lookups failed
        codes_by_code.value = codes_by_code.value || {};
        neighborhoods_by_id.value = neighborhoods_by_id.value || {};
    }

    // 2) compute bbox (use leaflet if available, otherwise map.bounds)
    let south = 44.883658, north = 45.008206, west = -93.217977, east = -92.993787;
    try {
        if (map.leaflet && typeof map.leaflet.getBounds === 'function') {
            const b = map.leaflet.getBounds();
            south = b.getSouth();
            north = b.getNorth();
            west = b.getWest();
            east = b.getEast();
        } else if (map.bounds) {
            south = map.bounds.se?.lat ?? south;
            north = map.bounds.nw?.lat ?? north;
            west = map.bounds.nw?.lng ?? west;
            east = map.bounds.se?.lng ?? east;
        }
    } catch (e) {
        console.warn('initializeCrimes: bbox calculation failed, using defaults', e);
    }

    // 3) request incidents (API may not support bbox filtering; do bbox filtering client-side)
    const params = new URLSearchParams({ limit: '1000' });
    const url = `${base}/incidents?${params.toString()}`;
    console.info('initializeCrimes: requesting', url);

    try {
        const { data: json } = await fetchJsonSafe(url);
        const incidents = Array.isArray(json) ? json : ((json && json.incidents) ? json.incidents : []);

        // 4) normalize rows and only keep incidents in the visible bbox
        const rows = incidents.map(i => {
            const codeKey = String(i.code ?? i.crime_code ?? '');
            const nbhKey = String(i.neighborhood_number ?? i.neighborhood ?? i.neighborhood_id ?? '');
            return {
                id: i.id ?? i.case_number ?? i.case ?? null,
                case_number: i.case_number ?? i.case ?? '',
                date: i.date ?? (i.date_time ? i.date_time.split(' ')[0] : ''),
                time: i.time ?? (i.date_time ? i.date_time.split(' ')[1] : ''),
                incident_type: codes_by_code.value[String(codeKey)] || (i.incident_type ?? i.incident ?? String(codeKey)),
                neighborhood_number: nbhKey,
                neighborhood_name: neighborhoods_by_id.value[nbhKey] || i.neighborhood_name || '',
                block: i.block ?? i.address ?? '',
                latitude: (i.latitude ?? i.lat) != null ? parseFloat(i.latitude ?? i.lat) : null,
                longitude: (i.longitude ?? i.lng) != null ? parseFloat(i.longitude ?? i.lng) : null,
                raw: i
            };
        }).filter(r => {
            // Only show crimes that occurred within the current visible bbox (defensive)
            if (r.latitude == null || r.longitude == null) return true;
            if (r.latitude != null && r.longitude != null && (isNaN(r.latitude) || isNaN(r.longitude))) return false;
            return r.latitude >= south && r.latitude <= north && r.longitude >= west && r.longitude <= east;
        });

        // 5) order with most recent on top
        rows.sort((a, b) => {
            const at = Date.parse(`${a.date}T${a.time || '00:00:00'}`);
            const bt = Date.parse(`${b.date}T${b.time || '00:00:00'}`);
            if (Number.isNaN(at) || Number.isNaN(bt)) return 0;
            return bt - at;
        });

        crimes.value = rows;

        // 6) compute per-neighborhood counts (keyed by neighborhood_number)
        const counts = {};
        rows.forEach(r => {
            const id = String(r.neighborhood_number ?? '');
            if (!id) return;
            counts[id] = (counts[id] || 0) + 1;
        });
        neighborhood_crime_counts.value = counts;

        // 7) draw one marker per neighborhood center and set popup with counts
        if (map.leaflet && Array.isArray(map.neighborhood_markers)) {
            map.neighborhood_markers.forEach((m, idx) => {
                const neighborhoodId = String(idx + 1);
                if (!m.marker) {
                    m.marker = L.marker(m.location).addTo(map.leaflet);
                }
                m.neighborhoodId = neighborhoodId;
                const name = neighborhoods_by_id.value[neighborhoodId] || `Neighborhood ${neighborhoodId}`;
                const cnt = counts[neighborhoodId] || 0;
                const popupHtml = `<div><strong>${name}</strong><br/>Crimes in view: ${cnt}</div>`;
                m.marker.bindPopup(popupHtml);
            });
        }

        console.info(`initializeCrimes: loaded ${rows.length} incidents`);
    } catch (err) {
        console.error('initializeCrimes error:', err);
        alert('Failed to load incidents: ' + (err.message || String(err)));
    }
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// Parse location input - checks if it's coordinates or an address
function parseLocationInput(input) {
    input = input.trim();
    
    // Check if input is coordinates (lat, lng or lat lng)
    const coordPattern = /^(-?\d+\.?\d*)\s*[,\s]\s*(-?\d+\.?\d*)$/;
    const match = input.match(coordPattern);
    
    if (match) {
        const lat = parseFloat(match[1]);
        const lng = parseFloat(match[2]);
        
        // Validate coordinate ranges
        if (lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180) {
            return { type: 'coordinates', lat, lng };
        }
    }
    
    // Otherwise treat as address
    return { type: 'address', query: input };
}

// Constrain coordinates to map bounds
function constrainToBounds(lat, lng) {
    // Use the bounds from map.bounds (nw and se corners)
    const minLat = map.bounds.se.lat;
    const maxLat = map.bounds.nw.lat;
    const minLng = map.bounds.nw.lng;
    const maxLng = map.bounds.se.lng;
    
    return {
        lat: Math.max(minLat, Math.min(lat, maxLat)),
        lng: Math.max(minLng, Math.min(lng, maxLng))
    };
}

// Geocode an address using Nominatim API
async function geocodeAddress(address) {
    try {
        // Add bounding box to limit search to St. Paul area (west, south, east, north)
        const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(address)}&limit=1&bounded=1&viewbox=-93.217977,45.008206,-92.993787,44.883658`;
        
        const response = await fetch(url, {
            headers: {
                'User-Agent': 'StPaulCrimeMap/1.0'
            }
        });
        
        if (!response.ok) {
            throw new Error(`Geocoding request failed with status ${response.status}`);
        }
        
        const data = await response.json();
        
        if (data && data.length > 0) {
            let lat = parseFloat(data[0].lat);
            let lng = parseFloat(data[0].lon);
            
            // Constrain to map bounds if map is initialized
            if (map.leaflet) {
                const constrained = constrainToBounds(lat, lng);
                lat = constrained.lat;
                lng = constrained.lng;
            }
            
            return {
                lat: lat,
                lng: lng,
                address: data[0].display_name
            };
        } else {
            throw new Error('Location not found. Please try a more specific address within St. Paul.');
        }
    } catch (error) {
        console.error('Geocoding error:', error);
        if (error.message.includes('Failed to fetch') || error.message.includes('NetworkError')) {
            throw new Error('Network error. Please check your internet connection and try again.');
        }
        throw error;
    }
}

// Reverse geocode coordinates to get address
async function reverseGeocode(lat, lng) {
    try {
        const response = await fetch(
            `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}&zoom=18&addressdetails=1`,
            {
                headers: {
                    'User-Agent': 'StPaulCrimeMap/1.0'
                }
            }
        );
        
        if (!response.ok) {
            throw new Error('Reverse geocoding request failed');
        }
        
        const data = await response.json();
        
        if (data && data.display_name) {
            return data.display_name;
        } else {
            return `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
        }
    } catch (error) {
        console.error('Reverse geocoding error:', error);
        // Fallback to coordinates if reverse geocoding fails
        return `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
    }
}

// Update map to new location
async function goToLocation() {
    if (!location_input.value.trim()) {
        return;
    }
    
    is_programmatic_move.value = true;
    user_has_interacted.value = false; // Reset interaction flag when user searches
    
    try {
        const parsed = parseLocationInput(location_input.value);
        let lat, lng, address;
        
        if (parsed.type === 'coordinates') {
            lat = parsed.lat;
            lng = parsed.lng;
            
            // Constrain to map bounds
            const constrained = constrainToBounds(lat, lng);
            lat = constrained.lat;
            lng = constrained.lng;
            
            address = await reverseGeocode(lat, lng);
        } else {
            const geocodeResult = await geocodeAddress(parsed.query);
            lat = geocodeResult.lat;
            lng = geocodeResult.lng;
            address = geocodeResult.address;
        }
        
        // Update map center and zoom in to show the location better
        // Use zoom level 15 for a closer view of the location
        const zoomLevel = 15;
        map.leaflet.setView([lat, lng], zoomLevel);
        map.center.lat = lat;
        map.center.lng = lng;
        map.center.address = address;
        map.zoom = zoomLevel; // Update the stored zoom level
        
        // Update input with the resolved address or coordinates
        location_input.value = address || `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
    } catch (error) {
        const errorMessage = error.message || 'Could not find location. Please try a different address or coordinates.';
        alert(`Error: ${errorMessage}`);
        console.error('Location error:', error);
        // Keep the input value so user can edit it
    } finally {
        // Reset flag after moveend event has been processed
        setTimeout(() => {
            is_programmatic_move.value = false;
        }, 300);
    }
}

// Update location input from current map center (only called after user manually pans/zooms)
async function updateLocationInputFromMap() {
    if (!map.leaflet) return;
    
    const center = map.leaflet.getCenter();
    const address = await reverseGeocode(center.lat, center.lng);
    location_input.value = address;
    map.center.lat = center.lat;
    map.center.lng = center.lng;
    map.center.address = address;
}

// Handle Enter key in location input
function handleLocationInputKeydown(event) {
    if (event.key === 'Enter') {
        goToLocation();
    }
}

// Validate incident form
function validateIncidentForm() {
    incident_form_errors.value = {};
    let isValid = true;
    
    if (!incident_form.date.trim()) {
        incident_form_errors.value.date = 'Date is required';
        isValid = false;
    }
    
    if (!incident_form.time.trim()) {
        incident_form_errors.value.time = 'Time is required';
        isValid = false;
    }
    
    if (!incident_form.address.trim()) {
        incident_form_errors.value.address = 'Address is required';
        isValid = false;
    }
    
    if (!incident_form.latitude.trim()) {
        incident_form_errors.value.latitude = 'Latitude is required';
        isValid = false;
    } else if (isNaN(parseFloat(incident_form.latitude)) || parseFloat(incident_form.latitude) < -90 || parseFloat(incident_form.latitude) > 90) {
        incident_form_errors.value.latitude = 'Latitude must be a valid number between -90 and 90';
        isValid = false;
    }
    
    if (!incident_form.longitude.trim()) {
        incident_form_errors.value.longitude = 'Longitude is required';
        isValid = false;
    } else if (isNaN(parseFloat(incident_form.longitude)) || parseFloat(incident_form.longitude) < -180 || parseFloat(incident_form.longitude) > 180) {
        incident_form_errors.value.longitude = 'Longitude must be a valid number between -180 and 180';
        isValid = false;
    }
    
    if (!incident_form.crime_code.trim()) {
        incident_form_errors.value.crime_code = 'Crime code is required';
        isValid = false;
    }
    
    if (!incident_form.description.trim()) {
        incident_form_errors.value.description = 'Description is required';
        isValid = false;
    }
    
    if (!incident_form.neighborhood.trim()) {
        incident_form_errors.value.neighborhood = 'Neighborhood is required';
        isValid = false;
    }
    
    return isValid;
}

// Submit incident form
async function submitIncident() {
    // Reset previous messages
    incident_form_success.value = false;
    incident_form_error_message.value = '';
    
    // Validate form
    if (!validateIncidentForm()) {
        incident_form_error_message.value = 'Please fill out all required fields correctly.';
        return;
    }
    
    // Check if API URL is set
    if (!crime_url.value) {
        incident_form_error_message.value = 'Please enter the REST API URL first.';
        return;
    }
    
    incident_form_submitting.value = true;
    
    try {
        // Prepare incident data mapped to server's expected shape
        const base = String(crime_url.value).replace(/\/$/, '');

        // Reverse lookup for neighborhood id (neighborhoods_by_id maps id -> name)
        const nameToId = {};
        Object.entries(neighborhoods_by_id.value || {}).forEach(([id, name]) => { nameToId[name] = id; });

        const neighborhood_number = nameToId[incident_form.neighborhood] || (incident_form.neighborhood && !isNaN(Number(incident_form.neighborhood)) ? String(incident_form.neighborhood) : '1');

        // Ensure required server fields exist. Generate a case_number if none provided.
        const case_number = incident_form.case_number || `cn-${Date.now()}`;

        const payload = {
            case_number: case_number,
            date: incident_form.date,
            time: incident_form.time,
            code: incident_form.crime_code,
            incident: incident_form.description,
            police_grid: incident_form.police_grid || 1,
            neighborhood_number: neighborhood_number,
            block: incident_form.address || ''
        };

        // Make PUT request to server's /new-incident endpoint
        const response = await fetch(`${base}/new-incident`, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(payload)
        });

        if (!response.ok) {
            const text = await response.text().catch(() => '');
            throw new Error(`Server error: ${response.status} ${response.statusText} ${text}`);
        }
        
        // Success - reset form and show success message
        incident_form_success.value = true;
        resetIncidentForm();
        
        // Hide success message after 5 seconds
        setTimeout(() => {
            incident_form_success.value = false;
        }, 5000);
        
    } catch (error) {
        console.error('Error submitting incident:', error);
        incident_form_error_message.value = `Error submitting incident: ${error.message}. Please try again.`;
    } finally {
        incident_form_submitting.value = false;
    }
}

// Reset incident form
function resetIncidentForm() {
    incident_form.date = '';
    incident_form.time = '';
    incident_form.address = '';
    incident_form.latitude = '';
    incident_form.longitude = '';
    incident_form.crime_code = '';
    incident_form.description = '';
    incident_form.neighborhood = '';
    incident_form_errors.value = {};
}

// Update form location from map center
function useMapLocationForForm() {
    if (map.leaflet) {
        const center = map.leaflet.getCenter();
        incident_form.latitude = center.lat.toFixed(6);
        incident_form.longitude = center.lng.toFixed(6);
        
        // Try to get address from map center if available
        if (map.center.address) {
            incident_form.address = map.center.address;
        }
    }
}

// Populate checkboxes from lookups (call this after you set codes_by_code.value and neighborhoods_by_id.value)
function populateFilterLists() {
    // incident types from codes_by_code
    const types = new Set(Object.values(codes_by_code.value || {}).filter(Boolean));
    types.forEach(t => { if (!(t in filters.incident_types)) filters.incident_types[t] = true; });

    // neighborhoods from neighborhoods_by_id
    Object.values(neighborhoods_by_id.value || {}).forEach(n => {
        if (n && !(n in filters.neighborhoods)) filters.neighborhoods[n] = true;
    });
}

// Build request using bbox + filters, fetch incidents, normalize, sort, update crimes and neighborhood counts
async function refreshCrimesWithFilters() {
    if (!crime_url || !crime_url.value) return;
    const base = String(crime_url.value).replace(/\/$/, '');

    // compute bbox (leaflet preferred)
    let south = 44.883658, north = 45.008206, west = -93.217977, east = -92.993787;
    try {
        if (map && map.leaflet && typeof map.leaflet.getBounds === 'function') {
            const b = map.leaflet.getBounds();
            south = b.getSouth(); north = b.getNorth(); west = b.getWest(); east = b.getEast();
        } else if (map && map.bounds) {
            south = map.bounds.se?.lat ?? south;
            north = map.bounds.nw?.lat ?? north;
            west = map.bounds.nw?.lng ?? west;
            east = map.bounds.se?.lng ?? east;
        }
    } catch (e) {
        console.warn('refreshCrimesWithFilters: bbox failed, using defaults', e);
    }

    const params = new URLSearchParams();
    params.append('limit', String(filters.max_incidents || 1000));
    params.append('lat_min', String(south));
    params.append('lat_max', String(north));
    params.append('lng_min', String(west));
    params.append('lng_max', String(east));
    if (filters.start_date) params.append('start_date', filters.start_date);
    if (filters.end_date) params.append('end_date', filters.end_date);

    // map selected incident type names back to codes (if lookups exist)
    const activeTypes = new Set(Object.entries(filters.incident_types).filter(([k,v]) => v).map(([k]) => k));
    const allTypes = new Set(Object.keys(filters.incident_types || {}));
    
    // Only filter by codes if some (but not all) types are selected
    // If all types are selected, don't send code filter (let backend return all)
    const selectedCodesSet = new Set();
    if (activeTypes.size > 0 && activeTypes.size < allTypes.size) {
        // Some types are unchecked, so filter by selected types
        Object.entries(codes_by_code.value || {}).forEach(([code, type]) => {
            if (activeTypes.has(type) && code != null && code !== '') {
                // Ensure code is a clean string and add to set to avoid duplicates
                selectedCodesSet.add(String(code).trim());
            }
        });
    }
    // If we have selected codes, add to params as comma-separated string
    if (selectedCodesSet.size > 0) {
        const selectedCodesArray = Array.from(selectedCodesSet).sort();
        params.append('code', selectedCodesArray.join(','));
    }

    // map selected neighborhood names to ids
    const nameToId = {};
    Object.entries(neighborhoods_by_id.value || {}).forEach(([id, name]) => { nameToId[name] = id; });
    const selectedNeighborhoodIds = Object.entries(filters.neighborhoods)
        .filter(([name, checked]) => checked && nameToId[name])
        .map(([name]) => nameToId[name]);
    if (selectedNeighborhoodIds.length) params.append('neighborhood', selectedNeighborhoodIds.join(','));
    console.log(params);
    console.log(params.toString());
    const url = `${base}/incidents?${params.toString()}`;
    console.info('refreshCrimesWithFilters requesting', url);

    try {
        const { data: json } = await fetchJsonSafe(url);
        const incidents = Array.isArray(json) ? json : ((json && json.incidents) ? json.incidents : []);

        const rows = incidents.map(i => {
            const codeKey = String(i.code ?? i.crime_code ?? '');
            const nbhKey = String(i.neighborhood_number ?? i.neighborhood ?? i.neighborhood_id ?? '');
            return {
                id: i.id ?? i.case_number ?? i.case ?? null,
                case_number: i.case_number ?? i.case ?? '',
                date: i.date ?? (i.date_time ? i.date_time.split(' ')[0] : ''),
                time: i.time ?? (i.date_time ? i.date_time.split(' ')[1] : ''),
                incident_type: (codes_by_code.value && codes_by_code.value[String(codeKey)]) || i.incident_type || i.incident || String(codeKey),
                neighborhood_number: nbhKey,
                neighborhood_name: (neighborhoods_by_id.value && neighborhoods_by_id.value[nbhKey]) || i.neighborhood_name || '',
                block: i.block ?? i.address ?? '',
                latitude: (i.latitude ?? i.lat) != null ? parseFloat(i.latitude ?? i.lat) : null,
                longitude: (i.longitude ?? i.lng) != null ? parseFloat(i.longitude ?? i.lng) : null,
                raw: i
            };
        }).filter(r => {
            if (r.latitude == null || r.longitude == null) return true;
            if (r.latitude != null && r.longitude != null && (isNaN(r.latitude) || isNaN(r.longitude))) return false;
            return r.latitude >= south && r.latitude <= north && r.longitude >= west && r.longitude <= east;
        });

        rows.sort((a, b) => {
            const at = Date.parse(`${a.date}T${a.time || '00:00:00'}`);
            const bt = Date.parse(`${b.date}T${b.time || '00:00:00'}`);
            if (Number.isNaN(at) || Number.isNaN(bt)) return 0;
            return bt - at;
        });

        crimes.value = rows;

        // compute neighborhood counts
        const counts = {};
        rows.forEach(r => {
            const id = String(r.neighborhood_number ?? '');
            if (!id) return;
            counts[id] = (counts[id] || 0) + 1;
        });
        neighborhood_crime_counts.value = counts;

        // update neighborhood markers/popups
        try { drawNeighborhoodMarkers(); updateNeighborhoodMarkerPopups(); } catch (e) { /* non-fatal */ }

        console.info(`refreshCrimesWithFilters: loaded ${rows.length} incidents`);
    } catch (err) {
        console.error('refreshCrimesWithFilters error:', err);
        alert('Failed to refresh incidents: ' + (err.message || String(err)));
    }
}

function resetFilters() {
    Object.keys(filters.incident_types).forEach(k => filters.incident_types[k] = true);
    Object.keys(filters.neighborhoods).forEach(k => filters.neighborhoods[k] = true);
    filters.start_date = '';
    filters.end_date = '';
    filters.max_incidents = 1000;
}
// Draw markers for neighborhoods (only for neighborhoods that have crimes or are in the lookup)
function drawNeighborhoodMarkers() {
    if (!map.leaflet || !Array.isArray(map.neighborhood_markers)) return;
    
    // Get all neighborhood IDs that either have crimes or exist in the lookup
    const relevantNeighborhoodIds = new Set();
    Object.keys(neighborhood_crime_counts.value).forEach(id => relevantNeighborhoodIds.add(id));
    Object.keys(neighborhoods_by_id.value).forEach(id => relevantNeighborhoodIds.add(id));
    
    // Draw markers for neighborhoods
    map.neighborhood_markers.forEach((m, idx) => {
        const neighborhoodId = String(idx + 1); // Assume sequential IDs for hardcoded positions
        
        // Only create marker if this neighborhood has crimes or exists in lookup
        if (relevantNeighborhoodIds.has(neighborhoodId)) {
            if (!m.marker) {
                m.marker = L.marker(m.location).addTo(map.leaflet);
            }
            m.neighborhoodId = neighborhoodId;
        }
    });
}

// Update popups on neighborhood markers with current crime counts
function updateNeighborhoodMarkerPopups() {
    if (!map.leaflet || !Array.isArray(map.neighborhood_markers)) return;
    const counts = neighborhood_crime_counts.value || {};
    
    map.neighborhood_markers.forEach((m) => {
        if (!m.marker || !m.neighborhoodId) return;
        
        const neighborhoodId = String(m.neighborhoodId);
        const name = neighborhoods_by_id.value[neighborhoodId] || `Neighborhood ${neighborhoodId}`;
        const cnt = counts[neighborhoodId] || 0;
        const popupHtml = `<div><strong>${name}</strong><br/>Crimes in view: ${cnt}</div>`;
        m.marker.bindPopup(popupHtml);
    });
}

// add these reactive vars (near your other refs)
let selected_crime_marker = ref(null);
let selected_crime_id = ref(null);

// sanitize obfuscated address numbers (replace X in the leading house number with '0')
function sanitizeAddressNumber(addr) {
    if (!addr || typeof addr !== 'string') return addr;
    // Replace single 'X' immediately after leading digits: "98X UNIVERSITY AV W" -> "980 UNIVERSITY AV W"
    return addr.replace(/^(\d+)X(?=\b| )/, '$10');
}

// categorize crimes (simple keyword-based)
function categorizeCrime(crime) {
    const text = String(crime.incident_type || crime.raw?.incident_type || crime.raw?.incident || '').toLowerCase();
    const violent = ['assault','battery','robbery','homicide','murder','rape','sexual','aggravated','threat'];
    const property = ['theft','burglary','larceny','vandalism','shoplifting','arson','vehicle','motor vehicle','auto','stolen'];
    for (const k of violent) if (text.includes(k)) return 'violent';
    for (const k of property) if (text.includes(k)) return 'property';
    return 'other';
}

function crimeRowClass(crime) {
    return 'crime-row ' + categorizeCrime(crime);
}

// show a distinct marker for a selected crime and attach popup with delete button
async function selectCrime(crime) {
    if (!map.leaflet) return;
    // remove previous selected marker
    if (selected_crime_marker.value) {
        selected_crime_marker.value.remove();
        selected_crime_marker.value = null;
        selected_crime_id.value = null;
    }

    let lat = parseFloat(crime.latitude ?? crime.lat ?? NaN);
    let lng = parseFloat(crime.longitude ?? crime.lng ?? NaN);

    if (!isFinite(lat) || !isFinite(lng)) {
        // try geocoding sanitized address
        const addr = sanitizeAddressNumber(crime.block || crime.address || crime.neighborhood_name || '');
        if (addr) {
            try {
                const geo = await geocodeAddress(addr);
                if (geo && geo.lat && geo.lon) {
                    lat = parseFloat(geo.lat);
                    lng = parseFloat(geo.lon);
                }
            } catch (e) {
                console.warn('Geocode failed', e);
            }
        }
    }

    if (!isFinite(lat) || !isFinite(lng)) {
        alert('No valid location available for this incident.');
        return;
    }

    // distinct style: red circle marker
    const marker = L.circleMarker([lat, lng], {
        color: '#b30000',
        fillColor: '#ff6666',
        fillOpacity: 0.9,
        radius: 7,
        weight: 2
    }).addTo(map.leaflet);

    const popupId = `delete-btn-${encodeURIComponent(String(crime.id ?? crime.case_number ?? Math.random()))}`;
    const popupHtml = `
        <div>
            <div><strong>${crime.date || ''} ${crime.time || ''}</strong></div>
            <div>${crime.incident_type || ''}</div>
            <div style="margin-top:.5rem;">
                <button id="${popupId}" style="background:#c82333;color:#fff;border:0;padding:.35rem .6rem;border-radius:4px;cursor:pointer;">Delete</button>
            </div>
        </div>
    `;
    marker.bindPopup(popupHtml);

    marker.on('popupopen', () => {
        const btn = document.getElementById(popupId);
        if (!btn) return;
        const handler = async (ev) => {
            ev.preventDefault();
            btn.disabled = true;
            try {
                await deleteIncident(crime);
            } catch (err) {
                console.error('Delete failed:', err);
                alert('Delete failed: ' + (err.message || err));
                btn.disabled = false;
            } finally {
                btn.removeEventListener('click', handler);
            }
        };
        btn.addEventListener('click', handler);
    });

    selected_crime_marker.value = marker;
    selected_crime_id.value = crime.id ?? crime.case_number ?? null;

    try { map.leaflet.setView([lat, lng], Math.max(map.leaflet.getZoom(), 15)); } catch {}
}

// delete incident via REST API and update UI
async function deleteIncident(crime) {
    if (!crime_url || !crime_url.value) throw new Error('API base URL not set');
    const id = crime.id ?? crime.case_number ?? crime.case ?? null;
    if (!id) throw new Error('No identifier for incident');

    const base = String(crime_url.value).replace(/\/$/, '');
    // try DELETE /incidents/:id first, fall back to /remove-incident
    const tryUrl = `${base}/incidents/${encodeURIComponent(String(id))}`;
    if (!confirm('Delete this incident from the database? This cannot be undone.')) return;

    let resp;
    try {
        resp = await fetch(tryUrl, { method: 'DELETE' });
        if (!resp.ok) {
            // fallback: call /remove-incident with JSON body
            const fallbackUrl = `${base}/remove-incident`;
            resp = await fetch(fallbackUrl, {
                method: 'DELETE',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ id: id, case_number: id })
            });
        }
    } catch (e) {
        throw new Error('Network error: ' + e.message);
    }

    if (!resp.ok) {
        const text = await resp.text().catch(() => '');
        throw new Error(`Delete failed (${resp.status}) ${text}`);
    }

    // remove from local list and update counts/popups
    crimes.value = crimes.value.filter(c => String(c.id ?? c.case_number ?? c.case ?? '') !== String(id));

    const counts = {};
    crimes.value.forEach(r => {
        const nid = String(r.neighborhood_number ?? '');
        if (!nid) return;
        counts[nid] = (counts[nid] || 0) + 1;
    });
    neighborhood_crime_counts.value = counts;

    // remove selected marker if it was for this crime
    if (selected_crime_id.value && String(selected_crime_id.value) === String(id)) {
        if (selected_crime_marker.value) {
            selected_crime_marker.value.remove();
            selected_crime_marker.value = null;
            selected_crime_id.value = null;
        }
    }

    // refresh neighborhood popups
    try { updateNeighborhoodMarkerPopups(); } catch (e) {}
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="grid-container">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
        <div class="grid-x grid-padding-x">
            <div class="cell small-12">
                <div class="location-search-container">
                    <input 
                        id="location-input" 
                        class="location-input" 
                        type="text" 
                        v-model="location_input" 
                        placeholder="Enter address or coordinates (lat, lng)"
                        @keydown="handleLocationInputKeydown"
                    />
                    <button class="button location-go-button" type="button" @click="goToLocation">Go</button>
                </div>
            </div>
        </div>

        <!-- Insert this filter UI block into your <template> near the location-search or above the incidents table -->
        <div class="grid-x grid-padding-x filter-container" style="padding:1rem; background:#fafafa; border-top:1px solid #eee;">
            <div class="cell small-12 medium-6">
                <h4 style="margin:0 0 .5rem 0">Incident Types</h4>
                <div style="max-height:200px; overflow:auto; border:1px solid #eee; padding:.5rem;">
                    <div v-for="(checked, type) in filters.incident_types" :key="type" style="margin-bottom:.25rem;">
                        <label style="cursor:pointer;">
                            <input type="checkbox" v-model="filters.incident_types[type]" /> {{ type }}
                        </label>
                    </div>
                </div>
            </div>

            <div class="cell small-12 medium-6">
                <h4 style="margin:0 0 .5rem 0">Neighborhoods</h4>
                <div style="max-height:200px; overflow:auto; border:1px solid #eee; padding:.5rem;">
                    <div v-for="(checked, name) in filters.neighborhoods" :key="name" style="margin-bottom:.25rem;">
                        <label style="cursor:pointer;">
                            <input type="checkbox" v-model="filters.neighborhoods[name]" /> {{ name }}
                        </label>
                    </div>
                </div>
            </div>

            <div class="cell small-12 medium-4" style="margin-top:.5rem;">
                <label>Start date</label>
                <input type="date" v-model="filters.start_date" />
            </div>
            <div class="cell small-12 medium-4" style="margin-top:.5rem;">
                <label>End date</label>
                <input type="date" v-model="filters.end_date" />
            </div>
            <div class="cell small-12 medium-4" style="margin-top:.5rem;">
                <label>Max incidents</label>
                <input type="number" min="1" step="1" v-model.number="filters.max_incidents" />
            </div>

            <div class="cell small-12" style="margin-top:.5rem;">
                <button class="button" type="button" @click="refreshCrimesWithFilters()">Update</button>
                <button class="button secondary" type="button" @click="resetFilters()">Reset</button>
            </div>
        </div>
        <!-- end filter UI -->

        <div class="grid-x grid-padding-x">
            <div class="cell small-12">
                            <!-- Crime Category Legend -->
                <div style="padding:1rem; background:#fff; border:1px solid #ddd; margin-bottom:1rem; border-radius:4px;">
                    <h4 style="margin:0 0 0.5rem 0;">Crime Category Legend</h4>
                    <div style="display:flex; gap:1.5rem; flex-wrap:wrap;">
                        <div style="display:flex; align-items:center; gap:0.5rem;">
                            <div style="width:20px; height:20px; background:#ffe9e6; border:1px solid #ccc;"></div>
                            <span>Violent Crimes</span>
                        </div>
                        <div style="display:flex; align-items:center; gap:0.5rem;">
                            <div style="width:20px; height:20px; background:#e6f0ff; border:1px solid #ccc;"></div>
                            <span>Property Crimes</span>
                        </div>
                        <div style="display:flex; align-items:center; gap:0.5rem;">
                            <div style="width:20px; height:20px; background:#f7f7f7; border:1px solid #ccc;"></div>
                            <span>Other Crimes</span>
                        </div>
                    </div>
                </div>
                
                <div class="incident-form-container">
                    <h2 class="incident-form-header">New Incident Upload Form</h2>
                    
                    <div v-if="incident_form_success" class="incident-form-message success">
                        Incident submitted successfully!
                    </div>
                    <div v-if="incident_form_error_message" class="incident-form-message error">
                        {{ incident_form_error_message }}
                    </div>
                    
                    <form @submit.prevent="submitIncident" class="incident-form">
                        <div class="grid-x grid-margin-x">
                            <div class="cell small-12 medium-6">
                                <label for="incident-date">Date <span class="required">*</span></label>
                                <input 
                                    id="incident-date" 
                                    type="date" 
                                    v-model="incident_form.date"
                                    :class="{ 'error': incident_form_errors.date }"
                                />
                                <span v-if="incident_form_errors.date" class="field-error">{{ incident_form_errors.date }}</span>
                            </div>
                            
                            <div class="cell small-12 medium-6">
                                <label for="incident-time">Time <span class="required">*</span></label>
                                <input 
                                    id="incident-time" 
                                    type="time" 
                                    v-model="incident_form.time"
                                    :class="{ 'error': incident_form_errors.time }"
                                />
                                <span v-if="incident_form_errors.time" class="field-error">{{ incident_form_errors.time }}</span>
                            </div>
                            
                            <div class="cell small-12">
                                <label for="incident-address">Address <span class="required">*</span></label>
                                <input 
                                    id="incident-address" 
                                    type="text" 
                                    v-model="incident_form.address"
                                    placeholder="Enter address"
                                    :class="{ 'error': incident_form_errors.address }"
                                />
                                <span v-if="incident_form_errors.address" class="field-error">{{ incident_form_errors.address }}</span>
                            </div>
                            
                            <div class="cell small-12 medium-6">
                                <label for="incident-latitude">Latitude <span class="required">*</span></label>
                                <input 
                                    id="incident-latitude" 
                                    type="number" 
                                    step="any"
                                    v-model="incident_form.latitude"
                                    placeholder="e.g., 44.955139"
                                    :class="{ 'error': incident_form_errors.latitude }"
                                />
                                <span v-if="incident_form_errors.latitude" class="field-error">{{ incident_form_errors.latitude }}</span>
                            </div>
                            
                            <div class="cell small-12 medium-6">
                                <label for="incident-longitude">Longitude <span class="required">*</span></label>
                                <input 
                                    id="incident-longitude" 
                                    type="number" 
                                    step="any"
                                    v-model="incident_form.longitude"
                                    placeholder="e.g., -93.102222"
                                    :class="{ 'error': incident_form_errors.longitude }"
                                />
                                <span v-if="incident_form_errors.longitude" class="field-error">{{ incident_form_errors.longitude }}</span>
                            </div>
                            
                            <div class="cell small-12">
                                <button type="button" class="button secondary small" @click="useMapLocationForForm">
                                    Use Current Map Location
                                </button>
                            </div>
                            
                            <div class="cell small-12 medium-6">
                                <label for="incident-crime-code">Crime Code <span class="required">*</span></label>
                                <input 
                                    id="incident-crime-code" 
                                    type="text" 
                                    v-model="incident_form.crime_code"
                                    placeholder="Enter crime code"
                                    :class="{ 'error': incident_form_errors.crime_code }"
                                />
                                <span v-if="incident_form_errors.crime_code" class="field-error">{{ incident_form_errors.crime_code }}</span>
                            </div>
                            
                            <div class="cell small-12 medium-6">
                                <label for="incident-neighborhood">Neighborhood <span class="required">*</span></label>
                                <input 
                                    id="incident-neighborhood" 
                                    type="text" 
                                    v-model="incident_form.neighborhood"
                                    placeholder="Enter neighborhood"
                                    :class="{ 'error': incident_form_errors.neighborhood }"
                                />
                                <span v-if="incident_form_errors.neighborhood" class="field-error">{{ incident_form_errors.neighborhood }}</span>
                            </div>
                            
                            <div class="cell small-12">
                                <label for="incident-description">Description <span class="required">*</span></label>
                                <textarea 
                                    id="incident-description" 
                                    v-model="incident_form.description"
                                    placeholder="Enter incident description"
                                    rows="4"
                                    :class="{ 'error': incident_form_errors.description }"
                                ></textarea>
                                <span v-if="incident_form_errors.description" class="field-error">{{ incident_form_errors.description }}</span>
                            </div>
                            
                            <div class="cell small-12">
                                <button 
                                    type="submit" 
                                    class="button submit-button"
                                    :disabled="incident_form_submitting"
                                >
                                    {{ incident_form_submitting ? 'Submitting...' : 'Submit Incident' }}
                                </button>
                                <button 
                                    type="button" 
                                    class="button secondary"
                                    @click="resetIncidentForm"
                                    :disabled="incident_form_submitting"
                                >
                                    Clear Form
                                </button>
                            </div>
                        </div>
                    </form>
                </div>
            </div>
        </div>

        <div class="grid-x grid-padding-x">
            <div class="cell small-12">
                <table class="crime-table" style="width:100%; border-collapse:collapse;">
                    <thead>
                        <tr style="background-color:#f2f2f2; font-weight:bold;">
                            <th style="padding:0.5rem; text-align:left;">Date</th>
                            <th style="padding:0.5rem; text-align:left;">Time</th>
                            <th style="padding:0.5rem; text-align:left;">Incident Type</th>
                            <th style="padding:0.5rem; text-align:left;">Neighborhood</th>
                            <th style="padding:0.5rem; text-align:left;">Block</th>
                            <th style="padding:0.5rem; text-align:left;">Latitude</th>
                            <th style="padding:0.5rem; text-align:left;">Longitude</th>
                            <th style="padding:0.5rem; text-align:left;">Select</th>
                            <th style="padding:0.5rem; text-align:left;">Delete</th>
                        </tr>
                    </thead>
                    <tbody>
                        <!-- replace/augment your incidents table row rendering with this -->
                        <tr v-for="crime in crimes" :key="crime.id ?? crime.case_number" :class="crimeRowClass(crime)">
                            <td style="padding:0.5rem;">{{ crime.date }}</td>
                            <td style="padding:0.5rem;">{{ crime.time }}</td>
                            <td style="padding:0.5rem;">{{ crime.incident_type }}</td>
                            <td style="padding:0.5rem;">{{ crime.neighborhood_name }}</td>
                            <td style="padding:0.5rem;">{{ crime.block }}</td>
                            <td style="padding:0.5rem;">{{ crime.latitude }}</td>
                            <td style="padding:0.5rem;">{{ crime.longitude }}</td>
                            <td style="padding:0.5rem;">
                                <button type="button" @click="selectCrime(crime)">Select</button>
                            </td>
                            <td style="padding:0.5rem;">
                                <button type="button" class="delete-btn" @click="deleteIncident(crime)">Delete</button>
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}

.location-search-container {
    display: flex;
    gap: 0.5rem;
    padding: 1rem;
    background-color: #f8f8f8;
    border-top: 1px solid #ddd;
}

.location-input {
    flex: 1;
    padding: 0.5rem;
    font-size: 1rem;
    border: 1px solid #ccc;
    border-radius: 4px;
}

.location-input:focus {
    outline: none;
    border-color: #1779ba;
    box-shadow: 0 0 0 2px rgba(23, 121, 186, 0.25);
}

.location-go-button {
    padding: 0.5rem 1.5rem;
    font-size: 1rem;
    white-space: nowrap;
}

.incident-form-container {
    padding: 1.5rem;
    background-color: #fff;
    border-top: 1px solid #ddd;
    margin-top: 1rem;
}

.incident-form-header {
    font-size: 1.5rem;
    font-weight: bold;
    margin-bottom: 1rem;
    color: #0a0a0a;
}

.incident-form-message {
    padding: 0.75rem 1rem;
    margin-bottom: 1rem;
    border-radius: 4px;
    font-size: 0.95rem;
}

.incident-form-message.success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.incident-form-message.error {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
}

.incident-form {
    margin-top: 1rem;
}

.incident-form label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 600;
    color: #0a0a0a;
}

.incident-form .required {
    color: #D32323;
}

.incident-form input,
.incident-form textarea {
    width: 100%;
    padding: 0.5rem;
    font-size: 1rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    margin-bottom: 0.25rem;
}

.incident-form input:focus,
.incident-form textarea:focus {
    outline: none;
    border-color: #1779ba;
    box-shadow: 0 0 0 2px rgba(23, 121, 186, 0.25);
}

.incident-form input.error,
.incident-form textarea.error {
    border-color: #D32323;
}

.field-error {
    display: block;
    color: #D32323;
    font-size: 0.875rem;
    margin-bottom: 0.75rem;   
}

.submit-button {
    margin-right: 0.5rem;
}

.submit-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
}

.crime-row { transition: background-color .15s; }
.crime-row.violent { background: #ffe9e6; }   /* light red */
.crime-row.property { background: #e6f0ff; }  /* light blue */
.crime-row.other { background: #f7f7f7; }     /* light gray */

.delete-btn {
    background: #c82333;
    color: #fff;
    border: 0;
    padding: .25rem .5rem;
    border-radius: 4px;
    cursor: pointer;
}
.delete-btn:disabled { opacity: .6; cursor: default; }
</style>
