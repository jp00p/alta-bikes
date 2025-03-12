<template>
  <div>
    <div id="map-container"></div>
  </div>
</template>

<script>
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

const portlandCenter = [-122.67419412732124, 45.52024728220982];
const mapStyle = {
  version: 8,
  sources: {
    'osm': {
      type: 'raster',
      tiles: ['https://a.tile.openstreetmap.org/{z}/{x}/{y}.png'],
      tileSize: 256,
      attribution: '© OpenStreetMap Contributors'
    }
  },
  layers: [
    {
      id: 'osm',
      type: 'raster',
      source: 'osm',
      minzoom: 0,
      maxzoom: 19
    }
  ]
};

export default {
  data() {
    return {
      map: null,
      style: mapStyle,
      center: [-122.67419412732124, 45.52024728220982], // portland

      facilityLabels: {
        'ABL': 'Advisory Bike Lane',
        'BBBL': 'Bike Lane Buffered by Bus Lane',
        'BL': 'Bike Lane',
        'BBL': 'Buffered Bike Lane',
        'ESR': 'Enhanced Shared Roadway',
        'LSB': 'Local Service Bikeway',
        'NG': 'Neighborhood Greenway',
        'PBL': 'Protected Bike Lane',
        'SBBL': 'Shared Bus-Bike Lane',
        'SIR': 'Separated in-roadway',
        'TRL': 'Off-Street Path/Trail'
      },

      facilityClasses: {
        'Class 1': ['NG', 'TRL'],
        'Class 2': ['ABL', 'BBBL', 'BL', 'BBL', 'SBBL'],
        'Class 3': ['ESR', 'LSB'],
        'Class 4': ['PBL', 'SIR']
      },

      classColors: {
        'Class 1': '#FBAD18',
        'Class 2': '#F0565F',
        'Class 3': '#3862AE',
        'Class 4': '#4BC0B1'
      }
    };
  },
  mounted() {
    this.initializeMap();
  },
  beforeUnmount() {
    if (this.map) {
      this.map.remove();
    }
  },
  methods: {
    initializeMap() {
      this.map = new maplibregl.Map({
        container: 'map-container',
        style: mapStyle,
        center: this.center,
        zoom: 13
      });

      this.map.addControl(new maplibregl.NavigationControl());

      this.map.on('load', () => {
        this.map.setPaintProperty('osm', 'raster-opacity', 0.25);
        this.loadBikeFacilities();
        this.createJeremyHouseMarker();
      });
    },

    loadBikeFacilities() {
      const bikeDataUrl = '/data/portland-bike-facilities.geojson'; // i couldn't find a way to access it from the portlandmaps server

      fetch(bikeDataUrl)
        .then(response => {
          if (!response.ok) {
            throw new Error(`Failed to load data: ${response.status} ${response.statusText}`);
          }
          return response.json();
        })
        .then(data => {
          console.log("Successfully loaded GeoJSON data:", data);

          // check if we have features
          if (!data.features || data.features.length === 0) {
            console.error("GeoJSON has no features");
            return;
          }

          // add data source
          this.map.addSource('bike-facilities', {
            type: 'geojson',
            data: data
          });

          const facilityTypeField = 'Facility';
          const statusField = 'Status';

          
          Object.entries(this.facilityClasses).forEach(([className, facilityCodes]) => {

            const filter = ['any'];
            facilityCodes.forEach(code => {
              filter.push(['==', ['get', facilityTypeField], code]);
            });

            // Existing facilities (solid lines)
            // Note: Status includes ACTIVE instead of EXISTING
            this.map.addLayer({
              id: `${className}-existing`,
              type: 'line',
              source: 'bike-facilities',
              layout: {
                'line-join': 'round',
                'line-cap': 'round'
              },
              paint: {
                'line-color': this.classColors[className],
                'line-width': 4, // Slightly thicker
                'line-opacity': 0.9,
                'line-blur': 0.5,
              },
              filter: ['all',
                filter,
                ['==', ['get', statusField], 'ACTIVE']
              ]
            });

            // Planned facilities (dashed lines)
            // Include both PLANNED and RECOMM (recommended) statuses
            this.map.addLayer({
              id: `${className}-planned`,
              type: 'line',
              source: 'bike-facilities',
              layout: {
                'line-join': 'round',
                'line-cap': 'round'
              },
              paint: {
                'line-color': this.classColors[className],
                'line-width': 3,
                'line-dasharray': [2, 2]
              },
              filter: ['all',
                filter,
                ['any',
                  ['==', ['get', statusField], 'PLANNED'],
                  ['==', ['get', statusField], 'RECOMM']
                ]
              ]
            });
          });

            // add popups with correct field references
            this.addPopups(facilityTypeField, statusField);

            // add cursor interaction
            this.addCursorInteraction();

            // auto-fit map to data bounds
            if (data.features && data.features.length > 0) {
              this.fitMapToBounds(data);
            }
        })
        .catch(error => {
          console.error('Error loading bike facility data:', error);
        });
    },

    
    findFacilityTypeField(props) {
      const possibilities = ['FACILITYTYPE', 'FACILITY_TYPE', 'TYPE', 'BIKETYPE', 'BIKE_TYPE', 'FACILITY'];
      for (const field of possibilities) {
        if (props[field] !== undefined) return field;
      }
      // return the first key as fallback
      return Object.keys(props)[0];
    },

    findStatusField(props) {
      const possibilities = ['STATUS', 'FACSTATUS', 'FAC_STATUS', 'STATE'];
      for (const field of possibilities) {
        if (props[field] !== undefined) return field;
      }
      return 'STATUS'; // default fallback
    },

    getFacilityClass(type) {
      for (const [className, facilities] of Object.entries(this.facilityClasses)) {
        if (facilities.includes(type)) {
          return className;
        }
      }
      return 'Unclassified';
    },

    getFacilityLabel(type) {
      return this.facilityLabels[type] || type;
    },

    // handle fitting map to bounds
    fitMapToBounds(data) {
      try {
        const bounds = new maplibregl.LngLatBounds();
        let boundsSet = false;

        data.features.forEach(feature => {
          if (feature.geometry && feature.geometry.coordinates) {
            if (feature.geometry.type === 'LineString') {
              // for LineString, each coordinate is a point
              feature.geometry.coordinates.forEach(coord => {
                bounds.extend(coord);
                boundsSet = true;
              });
            } else if (feature.geometry.type === 'MultiLineString') {
              // for MultiLineString, each coordinate is an array of points
              feature.geometry.coordinates.forEach(line => {
                line.forEach(coord => {
                  bounds.extend(coord);
                  boundsSet = true;
                });
              });
            } else if (feature.geometry.type === 'Point') {
              bounds.extend(feature.geometry.coordinates);
              boundsSet = true;
            }
          }
        });

        if (boundsSet) {
          this.map.fitBounds(bounds, { padding: 50 });
          this.map.setZoom(13)
        } else {
          console.warn('No valid coordinates found to fit bounds');
        }
      } catch (e) {
        // fallback to default center and zoom
        console.error("Error fitting bounds:", e);
        this.map.setCenter(this.center);
        this.map.setZoom(13);
      }
    },

    addPopups(facilityTypeField, statusField) {
      this.map.on('click', (e) => {
        const bikeLayerIds = [];
        Object.keys(this.facilityClasses).forEach(className => {
          bikeLayerIds.push(`${className}-existing`);
          bikeLayerIds.push(`${className}-planned`);
        });

        // only query features from bike facility layers
        const features = this.map.queryRenderedFeatures(e.point, {
          layers: bikeLayerIds
        });

        if (!features.length) return; // no bike facilities found at click point

        const feature = features[0];
        const facilityType = feature.properties[facilityTypeField];
        const status = feature.properties[statusField];
        const streetName = feature.properties.SegmentName || 'Unnamed';
        const facilityClass = this.getFacilityClass(facilityType);

        const popup = new maplibregl.Popup()
          .setLngLat(e.lngLat)
          .setHTML(`
              <h3>Bike Facility</h3>
              <p><strong>Type:</strong> ${this.facilityLabels[facilityType] || facilityType}</p>
              <p><strong>Class:</strong> ${facilityClass}</p>
              <p><strong>Street:</strong> ${streetName}</p>
              <p><strong>Status:</strong> ${status}</p>
          `)
          .addTo(this.map);
      });
    },

    addCursorInteraction() {
      const bikeLayerIds = [];
      Object.keys(this.facilityClasses).forEach(className => {
        bikeLayerIds.push(`${className}-existing`);
        bikeLayerIds.push(`${className}-planned`);
      });

      // change cursor to pointer when mouse enters a bike path
      bikeLayerIds.forEach(layerId => {
        this.map.on('mouseenter', layerId, () => {
          this.map.getCanvas().style.cursor = 'pointer';
        });

        // change cursor back to default when mouse leaves a bike path
        this.map.on('mouseleave', layerId, () => {
          this.map.getCanvas().style.cursor = '';
        });
      });
    },

    createJeremyHouseMarker() {
      // shameless plug
      const jeremyLngLat = [-122.664292, 45.648232]; 
      
      const el = document.createElement('div');
      el.className = 'custom-marker';
      el.style.width = '25px';
      el.style.height = '25px';
      
      new maplibregl.Marker(el)
        .setLngLat(jeremyLngLat)
        .setPopup(
          new maplibregl.Popup({ offset: 25 })
            .setHTML('<h3>Hire me! 🙏</h3>')
        )
        .addTo(this.map);
    }
  }
};
</script>

<style scoped>
  #map-container {
    width: 100%;
    height: 100vh;
    border-radius: 0;
    margin: 0;
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 0;
  }

</style>