<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script>
import mapboxgl from 'mapbox-gl';

export default {
  name: 'MyMap',
  props: {
    center: {
      type: Array,
      default: () => [139.7793490405977, 35.675065123476045]
    },
    zoom: {
      type: Number,
      default: 13
    }
  },
  mounted() {
    // 🔐 Set your Mapbox token here
    mapboxgl.accessToken = 'pk.eyJ1Ijoia3pob3U4IiwiYSI6ImNtYXFqdGRoNDAwcWQycHB0dTF4MWRrY2sifQ.6_LIbgwea8qHfJpFVdqC-A';

    this.map = new mapboxgl.Map({
      container: this.$refs.mapContainer,
      style: 'mapbox://styles/mapbox/light-v11', // ✅ Light gray style
      center: this.center,
      zoom: this.zoom
    });

    this.map.on('load', () => {
      this.$emit('map-loaded', this.map);

      // Add 'search-point' source
      this.map.addSource('search-point', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: []
        }
      });

      // Add symbol layer for label
      this.map.addLayer({
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
          'text-color': '#333',
          'text-halo-color': '#fff',
          'text-halo-width': 2
        }
      });

      // Add circle layer for point
      this.map.addLayer({
        id: 'search-point-layer',
        type: 'circle',
        source: 'search-point',
        paint: {
          'circle-radius': 8,
          'circle-color': '#ff7f00',
          'circle-stroke-width': 2,
          'circle-stroke-color': '#fff'
        }
      });
    });


  }
};

  
</script>

<style scoped>
.map-container {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
}
</style>
