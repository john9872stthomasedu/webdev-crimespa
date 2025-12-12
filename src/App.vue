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


// FUNCTIONS
// Function called once user has entered REST API URL
async function initializeCrimes() {
    if (!crime_url.value) {
        console.warn('initializeCrimes: crime_url not set');
        return;
    }

    const base = String(crime_url.value).replace(/\/$/, '');
    try {
        // Fetch lookups 
        let codesArr = [];
        let nbhArr = [];
        try {
            const [codesResp, nbhResp] = await Promise.all([
                fetch(`${base}/codes`),
                fetch(`${base}/neighborhoods`)
            ]);
            if (codesResp.ok) codesArr = await codesResp.json() || [];
            else console.warn('/codes returned', codesResp.status);

            if (nbhResp.ok) nbhArr = await nbhResp.json() || [];
            else console.warn('/neighborhoods returned', nbhResp.status);
        } catch (e) {
            console.warn('Lookup fetch error (continuing):', e);
        }

        // Build lookup maps
        const codeMap = {};
        if (Array.isArray(codesArr)) {
            codesArr.forEach(c => {
                if (c && (c.code != null)) codeMap[String(c.code)] = c.incident_type ?? c.incident_type_description ?? c.incident ?? '';
            });
        }

        const nbhMap = {};
        if (Array.isArray(nbhArr)) {
            nbhArr.forEach(n => {
                const id = n.id ?? n.neighborhood_number ?? n.neighborhood_id;
                const name = n.neighborhood_name ?? n.name;
                if (id != null) nbhMap[String(id)] = name ?? '';
            });
        }

        codes_by_code.value = codeMap;
        neighborhoods_by_id.value = nbhMap;

        // Compute bbox safely 
        let lat_min = 44.90, lat_max = 45.01, lng_min = -93.25, lng_max = -92.99;
        try {
            if (map && map.leaflet && typeof map.leaflet.getBounds === 'function') {
                const b = map.leaflet.getBounds();
                lat_min = b.getSouth();
                lat_max = b.getNorth();
                lng_min = b.getWest();
                lng_max = b.getEast();
            } else if (map && map.bounds) {
                // older fallback shape in your code
                lat_min = map.bounds.se?.lat ?? lat_min;
                lat_max = map.bounds.nw?.lat ?? lat_max;
                lng_min = map.bounds.nw?.lng ?? lng_min;
                lng_max = map.bounds.se?.lng ?? lng_max;
            }
        } catch (e) {
            console.warn('BBox calc failed, using defaults:', e);
        }

        // Request incidents 
        const params = new URLSearchParams({
            limit: '1000',
            lat_min: String(lat_min),
            lat_max: String(lat_max),
            lng_min: String(lng_min),
            lng_max: String(lng_max)
        });

        const incidentsResp = await fetch(`${base}/incidents?${params.toString()}`);
        if (!incidentsResp.ok) {
            throw new Error(`/incidents returned status ${incidentsResp.status}`);
        }

        const incidentsJson = await incidentsResp.json();
        const incidents = Array.isArray(incidentsJson) ? incidentsJson : (incidentsJson.incidents || []);

        // Normalize rows
        const rows = incidents.map(i => {
            const codeKey = String(i.code ?? i.crime_code ?? '');
            const nbhKey = String(i.neighborhood_number ?? i.neighborhood ?? i.neighborhood_id ?? '');
            return {
                id: i.id ?? i.case_number ?? i.case ?? null,
                case_number: i.case_number ?? i.case ?? '',
                date: i.date ?? '',
                time: i.time ?? '',
                incident_type: codeMap[codeKey] || i.incident_type || i.incident || codeKey,
                neighborhood_number: nbhKey,
                neighborhood_name: nbhMap[nbhKey] || i.neighborhood_name || '',
                block: i.block ?? i.address ?? '',
                latitude: i.latitude ?? i.lat ?? null,
                longitude: i.longitude ?? i.lng ?? null,
                raw: i
            };
        });

        // Sort newest first 
        rows.sort((a, b) => {
            const at = Date.parse(`${a.date}T${a.time || '00:00:00'}`);
            const bt = Date.parse(`${b.date}T${b.time || '00:00:00'}`);
            if (Number.isNaN(at) || Number.isNaN(bt)) return 0;
            return bt - at;
        });

        crimes.value = rows;

        // Counts per neighborhood
        const counts = {};
        rows.forEach(r => {
            const id = String(r.neighborhood_number ?? '');
            if (!id) return;
            counts[id] = (counts[id] || 0) + 1;
        });
        neighborhood_crime_counts.value = counts;

        // Draw/update markers safely
        try {
            drawNeighborhoodMarkers();
            updateNeighborhoodMarkerPopups();
        } catch (e) {
            console.warn('Marker draw/update failed (non-fatal):', e);
        }

        console.info(`initializeCrimes: loaded ${rows.length} incidents`);
    } catch (err) {
        console.error('initializeCrimes error:', err);
        // don't rethrow — surface a friendly alert and keep the UI alive
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
        // Prepare incident data
        const incidentData = {
            date: incident_form.date,
            time: incident_form.time,
            address: incident_form.address,
            latitude: parseFloat(incident_form.latitude),
            longitude: parseFloat(incident_form.longitude),
            crime_code: incident_form.crime_code,
            description: incident_form.description,
            neighborhood: incident_form.neighborhood
        };
        
        // Make PUT request
        const response = await fetch(crime_url.value, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(incidentData)
        });
        
        if (!response.ok) {
            throw new Error(`Server error: ${response.status} ${response.statusText}`);
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
        <div class="grid-x grid-padding-x">
            <div class="cell small-12">
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
</style>
