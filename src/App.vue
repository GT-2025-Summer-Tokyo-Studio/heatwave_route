<template>
  <div class="page-wrapper">

    <!-- Header always on top -->
    <header class="app-header">
      <h1>Interactive Heat-Responsive Shelter Routing Tool for Nihonbashi</h1>
      <p>Plan your route to the nearest shelter while minimizing heat risk.</p>
    </header>

    <!-- Intro Button -->
    <button class="intro-toggle" @click="showIntro = !showIntro">
      {{ showIntro ? 'Close' : 'Intro / Guide' }}
    </button>

    <!-- Slide-in Intro Panel -->
    <div v-if="showIntro" class="intro-panel">
      <h2>About This App</h2>
      <p>This tool helps you find the 5 nearest shelters based on your location while minimizing heat exposure.</p>
      <h3>How to Use:</h3>
      <ul>
        <li>Enter your location or station name.</li>
        <li>Click “Search” to find nearby shelters.</li>
        <li>Select one to view safe walking/driving routes.</li>
        <li>Click “Reset” to start over.</li>
      </ul>
    </div>


    <!-- Main map and sidebar area -->
    <div class="main-container">

      <!-- Sidebar -->
      <div class="search-container">
        <p><strong>Select your location</strong> to find the 5 closest shelters.</p>

        <div class="search-form">
          <input v-model="searchQuery" @keyup.enter="searchShelters" placeholder="Search for shelters..."
            class="search-input">
          <button @click="searchShelters" class="search-button">Search</button>
          <button @click="resetSearch" class="reset-button">Reset</button>
        </div>

        <!-- Shelter Results -->
        <div class="results-container" v-if="searchResults.length > 0">
          <h3>Nearest Shelters</h3>
          <ul class="results-list">
            <li
              v-for="shelter in searchResults"
              :key="shelter.id"
              class="shelter-item"
              :class="{ selected: shelter.id === selectedShelterId }"
              @click="selectShelter(shelter)"
            >
              <div class="shelter-name">{{ shelter.id }}. {{ shelter.name }}</div>
              <div class="shelter-details">
                <span>Distance: {{ shelter.distance.toFixed(0) }}m</span>
                <span>Capacity: {{ shelter.capacity }}</span>
              </div>
            </li>
          </ul>
        </div>
      </div>

      <!-- Route Summary -->
      <div class="route-summary-box" v-if="routeSummary">
        <h3>Route Summary</h3>
        <p><strong>Mode:</strong> {{ routeSummary.mode }}</p>
        <p><strong>Distance:</strong> {{ routeSummary.distance }} m</p>
        <p><strong>Estimated Time:</strong> {{ routeSummary.time }} min</p>
        <p><strong>Heat Exposure:</strong> {{ routeSummary.heatExposure }}</p>
        <p><strong>Heat Hazard:</strong> {{ routeSummary.heatHazard }}</p>
        <p><strong>Heat Scenario:</strong> {{ routeSummary.heatScenario }}</p>
        <p><strong>Vulnerability:</strong> {{ routeSummary.vulnerability }}</p>
        <p><strong>Max Vulnerability Level:</strong> {{ routeSummary.maxVulnerability }}</p>
        <p><strong>Water Needed:</strong> {{ routeSummary.water }} L</p>
      </div>

      <!-- Map -->
      <my-map :center="[139.777208, 35.683530]" :zoom="14.1" @map-loaded="mapLoaded"
        ref="mapComponent"></my-map>
    </div>
  </div>
</template>



<script>
import MyMap from './components/MyMap.vue'
import * as turf from '@turf/turf'

export default {
  components: {
    MyMap
  },
  data() {
    return {
      searchQuery: '',
      searchResults: [],
      map: null,
      popup: null,
      routeSummary: null,
      selectedShelterId: null,
      showIntro: true,
    }
  },
  methods: {
    addLegendItem(label, color, shape = 'line') {
      const legend = document.querySelector('.legend');
      if (!legend) return;

      // Avoid duplicate entries
      if ([...legend.querySelectorAll('div')].some(d => d.textContent === label)) return;

      const item = document.createElement('div');
      const symbol = document.createElement('span');

      if (shape === 'line') {
        symbol.style.width = '20px';
        symbol.style.height = '2px';
        symbol.style.background = color;
      } else if (shape === 'circle') {
        symbol.style.width = '10px';
        symbol.style.height = '10px';
        symbol.style.borderRadius = '50%';
        symbol.style.background = color;
      }

      symbol.style.display = 'inline-block';
      symbol.style.marginRight = '6px';

      item.appendChild(symbol);
      item.appendChild(document.createTextNode(label));
      legend.appendChild(item);
    },

    resetSearch() {
      // Reset data
      this.searchQuery = '';
      this.searchResults = [];
      this.selectedShelterId = null;
      this.routeSummary = null;

        const emptyFeatureCollection = {
          type: 'FeatureCollection',
          features: []
        };

      // Remove all related layers and sources if they exist
      const sources = [
        'search-point', 'search-buffer',
        'shelters-h', 'routes-points',
        'routes'
      ];

      sources.forEach(id => {
        if (map.getSource(id)) {
          map.getSource(id).setData(emptyFeatureCollection);
        }
      });

      //  reset the map view to default
      map.flyTo({
        center: [139.777208, 35.683530], 
        zoom: 14.1
      });
    },


    mapLoaded(map) {
      const that = this;
      this.map = map;
      window.map = map;

      // Inject legend container with class
      const legend = document.createElement('div');
      legend.className = 'legend';
      legend.innerHTML = '<h4>Legend</h4>';
      map.getContainer().appendChild(legend);
      console.log('Legend element:', legend);


      // Add districts
      map.addSource('districts', {
        type: 'geojson',
        data: import.meta.env.BASE_URL + 'data/districts.geojson'
      });
      map.addLayer({
        id: 'districts-line',
        type: 'line',
        source: 'districts',
        paint: {
          'line-color': '#FFA500',
          'line-opacity': 0.5,
          'line-width': 5
        }
      });

      this.addLegendItem('District Boundary', '#FFA500', 'line');
      


      // Add routes
      const arrow = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAnElEQVQ4T63TsQ0CMQyF4f/NgMQQ0CBR0FIx190cFIiWhhFoKdgEiRUeSoF0gO8cjkub+Evs2OLPpc9422dgBRwlNZn/BtjeApdOUJsh0QuuwKYWiYAFcAKWHaSR1EbpfAHlkO0ICdMJgV+QXmAAmUu6v9IZA8wkPVKgtg7TF7H25t4UbN+A9ahGmqqVD8AO2GdzUF45+I3ZJJb9JxbwRhEhB66xAAAAAElFTkSuQmCC'
      map.loadImage(arrow, function (error, image) {
        if (error) throw error
        map.addImage('route-arrow', image)
        map.addSource('routes', {
          type: 'geojson',
          data: turf.featureCollection([])
        })
        map.addSource('routes-points', {
          type: 'geojson',
          data: turf.featureCollection([])
        })
        map.addLayer({
          id: 'route-line',
          type: 'line',
          source: 'routes',
          layout: {
            'line-join': 'round',
            'line-cap': 'round'
          },
          paint: {
            'line-color': [
              'match',
              ['get', 'mode'],
              'drive',
              '#056113',
              '#80d18c'
            ],
            'line-width': 8
          }
        })

        that.addLegendItem('Driving Route', '#056113', 'line');
        that.addLegendItem('Walking Route', '#80d18c', 'line');




        map.addLayer({
          id: 'route-line-arror',
          source: 'routes',
          type: 'symbol',
          layout: {
            'symbol-placement': 'line',
            'symbol-spacing': 50,
            'icon-image': 'route-arrow',
            'icon-size': 0.6,
            'icon-allow-overlap': true,
          },
        })
        const color = [
          'match',
          ['get', 'type'],
          'start',
          '#67c23a',
          '#ff0000'
        ]
        map.addLayer({
          id: 'routes-points',
          type: 'circle',
          source: 'routes-points',
          paint: {
            'circle-radius': 8,
            'circle-color': color,
            'circle-opacity': 0.8,
            'circle-stroke-color': color,
            'circle-stroke-width': 2,
            'circle-stroke-opacity': 1
          }
        })

        that.addLegendItem('Start Point', '#67c23a', 'circle');
        that.addLegendItem('End Point', '#ff0000', 'circle');


        // add map evtent
        map.on('mousemove', 'route-line', e => {
          map.getCanvas().style.cursor = 'pointer'
        })
        map.on('mouseout', 'route-line', e => {
          map.getCanvas().style.cursor = ''
        })
        map.on('click', 'route-line', e => {
          const feature = e.features[0]
          const properties = feature.properties
          if (properties.mode === 'walk') that.showProperties(e.lngLat, properties)
        })
      });


      map.addSource('search-point', {
                  type: 'geojson',
                  data: turf.featureCollection([])
                })

      map.addLayer({
          id: 'search-point',
          type: 'circle',
          source: 'search-point',
          paint: {
            'circle-radius': 7,
            'circle-color': '#67c23a',
            'circle-opacity': 1,
            'circle-stroke-color': '#67c',
            'circle-stroke-width': 2
          }
        });

      this.addLegendItem('Search Point', '#67c23a', 'circle');

      map.addLayer({
        id: 'search-point-label',
        type: 'symbol',
        source: 'search-point',
        layout: {
          'text-field': ['get', 'name'],
          'text-offset': [0, 1.5],
          'text-anchor': 'top',
          'text-size': 14
        },
        paint: {
          'text-color': '#e6194b'
        }
      });

      map.addSource('search-buffer', {
        type: 'geojson',
        data: turf.featureCollection([])
      });

      map.addLayer({
        id: 'search-buffer-fill',
        type: 'fill',
        source: 'search-buffer',
        paint: {
          'fill-color': '#0080ff',
          'fill-opacity': 0.1
        }
      });

      map.addLayer({
        id: 'search-buffer-outline',
        type: 'line',
        source: 'search-buffer',
        paint: {
          'line-color': '#0077b6',
          'line-width': 2
        }
      });



      // Add shelters
      map.addSource('shelters', {
        type: 'geojson',
        data: import.meta.env.BASE_URL + 'data/shelters.geojson'
      })
      map.addSource('shelters-h', {
        type: 'geojson',
        data: turf.featureCollection([])
      })
      map.addLayer({
        id: 'shelters',
        type: 'circle',
        source: 'shelters',
        paint: {
          'circle-radius': 10,
          'circle-color': '#00AA00',
          'circle-opacity': 0.8,
          'circle-stroke-color': '#003366',
          'circle-stroke-width': 5,
          'circle-stroke-opacity': 1
        }
      })
      this.addLegendItem('Shelters', '#00f', 'circle');

      map.addLayer({
        id: 'shelters-h',
        type: 'circle',
        source: 'shelters-h',
        paint: {
          'circle-radius': 5,
          'circle-color': '#ff0',
          'circle-opacity': 0.8,
          'circle-stroke-color': '#ff0',
          'circle-stroke-width': 3,
          'circle-stroke-opacity': 0.4
        }
      })
      this.addLegendItem('Nearest 5 Shelters', '#ff0', 'circle');

      map.addLayer({
        id: 'shelters-label',
        type: 'symbol',
        source: 'shelters',
        layout: {
          'text-field': '{Name}',
          'text-font': ['Open Sans Semibold', 'Arial Unicode MS Bold'],
          'text-size': 14,
          'text-offset': [0, 1.5]
        },
        paint: {
          'text-color': '#000',
          'text-halo-color': '#fff',
          'text-halo-width': 1
        }
      })
      
    },
    showProperties(lngLat, properties) {
      const fields = Object.keys(properties);
      const content = fields.map(field => {
        const fieldName = field.split('_').map(v =>
          v.charAt(0).toUpperCase() + v.slice(1).toLowerCase()
        ).join(' ')
        return `<div class="field-item"><b class="field-label">${fieldName}：</b>${properties[field]}</div>`
      }).join('')
      this.popup = new mapboxgl.Popup({
        offset: [0, -5],
        anchor: "bottom",
        className: "my-popup",
        closeButton: true,
        maxWidth: "700px"
      })
        .setLngLat(lngLat)
        .setHTML(content)
        .addTo(map);
    },

    
    async searchShelters() {
      if (!this.searchQuery.trim()) return;

      try {
        const url = `https://flask-server-p724.onrender.com/query-shelters?address=${this.searchQuery}`;
        const response = await fetch(url);
        const mockResponse = await response.json();

        if (response.status === 200 && mockResponse.data?.shelters) {

          this.searchResults = mockResponse.data.shelters;

          const features = this.searchResults.map(result =>
            turf.point([result.longitude, result.latitude], result)
          );
          map.getSource('shelters-h').setData(turf.featureCollection(features));

          const bbox = turf.bbox(turf.featureCollection(features));
          map.fitBounds(bbox, { padding: 50 });


          const searchPointFeature = turf.point(
            [mockResponse.data.search_lon, mockResponse.data.search_lat],
            { name: this.searchQuery }
          );

          map.getSource('search-point').setData(
            turf.featureCollection([searchPointFeature])
          );

          // Create a 0.1 km (100 meter) buffer polygon around the search point
          const bufferPolygon = turf.buffer(searchPointFeature, 0.1, { units: 'kilometers' });

          // Add buffer to map
          map.getSource('search-buffer').setData(bufferPolygon);

        }
      } catch (error) {
        console.error('Error searching shelters:', error);
      }
    },

    async selectShelter(shelter) {
      this.selectedShelterId = shelter.id;
      if (this.popup) this.popup.remove()

      try {
        const url = `https://flask-server-p724.onrender.com/query-routes?address=${this.searchQuery}&shelter_id=${shelter.id}`
        // const url = `/data/mock-routes.json?q=${this.searchQuery}&id=${shelter.id}`
        const mockResponse = await fetch(url).then(res => res.json())

        if (mockResponse.status_code === 200) {
          const {
            drive_distance, drive_path_segments, drive_time, end_point, start_point, walk_path_segments, total_heat_risk, walk_distance, estimated_time_min, heat_exposure, heat_hazard, heat_risk_reference, heat_scenario, heat_vulnerability, max_vulnerability_level, water_needed_liters
          } = mockResponse.data
          // show route
          let routePaths = []
          drive_path_segments.forEach(segment => {
            routePaths.push({
              type: 'Feature',
              geometry: segment.geometry,
              properties: {
                mode: 'drive',
                drive_distance,
                drive_time,
              },
            })
          })
          walk_path_segments.forEach(segment => {
            routePaths.push({
              type: 'Feature',
              geometry: segment.geometry,
              properties: {
                mode: 'walk',
                total_heat_risk,
                walk_distance,
                estimated_time_min,
                heat_exposure,
                heat_hazard,
                heat_risk_reference,
                heat_scenario,
                heat_vulnerability,
                max_vulnerability_level,
                water_needed_liters
              },
            })
          })
          map.getSource('routes').setData(turf.featureCollection(routePaths))

          this.routeSummary = {
            mode: 'walk',
            distance: walk_distance,
            time: estimated_time_min,
            heatExposure: heat_exposure,
            heatHazard: heat_hazard,
            heatScenario: heat_scenario,
            vulnerability: heat_vulnerability,
            maxVulnerability: max_vulnerability_level,
            water: water_needed_liters
          };


          // set start and end points
          const routePoints = [
            {
              type: 'Feature',
              geometry: start_point,
              properties: {
                type: 'start',
                name: this.searchQuery
              },
            },
            {
              type: 'Feature',
              geometry: end_point,
              properties: {
                type: 'end',
                name: shelter.name
              },
            }
          ]
          map.getSource('routes-points').setData(turf.featureCollection(routePoints))
          const bbox = turf.bbox(turf.featureCollection(routePoints));
          map.fitBounds(bbox, { padding: 400 });
        }
      } catch (error) {
        console.error('Error selecting shelter:', error);
      }
    },
  }
}
</script>

<style lang="scss" scoped>
.page-wrapper {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  background: #d7d4d4;
  padding: 2rem;
  border-bottom: 1px solid #090909;
  z-index: 1001;

  h1 {
    margin: 0;
    font-size: 36px;
  }

  p {
    margin: 4px 0 0;
    color: #555;
    font-size: 24px
  }
}

.main-container {
  flex: 1;
  position: relative;
  overflow: hidden;
}

.search-container {
  position: absolute;
  top: 4rem;
  left: 1rem;
  z-index: 1000;
  background: rgb(172, 170, 169);
  padding: 28px;
  border-radius: 4px;
  width: 600px;

  .search-form {
    display: flex;
    gap: 10px;
    margin-bottom: 10px;
    font-size: 15px;
  }
}

.search-input {
  padding: 8px;
  border: 1px solid #fc0505;
  border-radius: 4px;
  flex-grow: 1;
  font-size: 20px;
}

.search-button {
  padding: 16px 24px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 15px;

  &:hover {
    background-color: #45a049;
  }
}

.results-container {
  background: rgb(245, 242, 242);
  padding: 24px;
  border-radius: 4px;
  box-shadow: 0 5px 10px rgba(122, 170, 2, 0.2);
  max-height: 60vh;
  overflow-y: auto;
  width: 100%;
  box-sizing: border-box;

  h3 {
    margin-top: 0;
    margin-bottom: 10px;
    color: #333;
    border-bottom: 1px solid #eee;
    padding-bottom: 0.8rem;
  }
}

.results-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.shelter-item {
  padding: 10px 0;
  border-bottom: 1px solid #eee;
  cursor: pointer;

  &:last-child {
    border-bottom: none;
  }
}

.shelter-name {
  font-weight: bold;
  margin-bottom: 5px;
  font-size: 18px;
}

.shelter-details {
  display: flex;
  justify-content: space-between;
  color: #a20404;
  font-size: 0.9em;
  font-size: 15px;
}

.selected {
  background-color: #e6f7ff;
  border-left: 4px solid #409eff;
  padding-left: 6px;
  font-size: 15px;
}

/* Legend: moved to top-right */
:deep(.legend) {
  position: absolute;
  top: 6rem;
  right: 1rem;
  background: white;
  padding: 20px;
  border-radius: 6px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.25);
  font-size: 16px;
  line-height: 22px;
  z-index: 1000;
  max-width: 280px;

  h4 {
    margin: 0 0 10px;
    font-size: 18px;
    font-weight: bold;
  }

  div {
    margin-bottom: 8px;
    display: flex;
    align-items: center;
  }

  span {
    display: inline-block;
    margin-right: 10px;
    flex-shrink: 0;
  }
}

/* Route Summary: larger text */
:deep(.route-summary-box) {
  position: absolute;
  bottom: 2rem;
  right: 1rem;
  width: 300px;
  background: white;
  padding: 20px;
  border-radius: 6px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.25);
  z-index: 1000;
  font-size: 15px;

  h3 {
    margin-top: 0;
    font-size: 18px;
    border-bottom: 1px solid #ccc;
    padding-bottom: 4px;
    margin-bottom: 8px;
  }

  p {
    margin: 6px 0;
    line-height: 1.5;
  }
}

.reset-button {
  margin-left: 10px;
  background-color: #f44336;
  color: white;
  padding: 6px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.intro-toggle {
  position: absolute;
  bottom: 18rem; /* below search box */
  left: 8rem;
  background-color: #1e293b;
  color: #fff;
  font-size: 14px;
  padding: 6px 12px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  z-index: 1001;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.intro-panel {
  position: absolute;
  bottom: 1rem;
  left: 8rem;
  width: 500px;
  background-color: white;
  padding: 16px;
  border-radius: 8px;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.15);
  z-index: 800;
  font-size: 14.5px;
  line-height: 1.5;
}

.intro-panel h3 {
  margin-top: 0;
  font-size: 18px;
  color: #0f172a;
}

.intro-panel h4 {
  margin: 1rem 0 0.5rem;
  font-size: 15px;
  color: #334155;
}

.intro-panel ul {
  padding-left: 1.2rem;
  margin: 0;
}

.intro-panel .note {
  margin-top: 1rem;
  font-size: 12px;
  color: #6b7280;
  font-style: italic;
}



</style>
