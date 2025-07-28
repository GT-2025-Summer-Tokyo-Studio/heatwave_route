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
      style: 'mapbox://styles/mapbox/streets-v12', // ✅ Light gray style
      center: this.center,
      zoom: this.zoom
    });

    this.map.on('load', () => {
      this.$emit('map-loaded', this.map);

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
