<template>
  <div class="map-container" ref="mapContainer">
    <slot></slot>
    <div class="map-tools">
      <slot name="tools"></slot>
    </div>
  </div>
</template>

<script>

export default {
  name: 'MyMap',
  props: {
    center: {
      type: Array,
      default: () => [116.4, 39.9]
    },
    zoom: {
      type: Number,
      default: 3
    },
    maxZoom: {
      type: Number,
      default: 17.35
    },
    minZoom: {
      type: Number,
      default: 2
    },
    mapStyle: {
      type: String,
      default: 'https://demotiles.maplibre.org/style.json'
      // default: '/data/style.json'
    },
  },
  data() {
    return {
    }
  },
  mounted() {
    this.initMap()
  },
  methods: {
    initMap() {
      fetch(this.mapStyle).then(res => res.json()).then(style => {
        const map = new mapboxgl.Map({
          container: this.$refs.mapContainer,
          maxZoom: this.maxZoom,
          minZoom: this.minZoom,
          zoom: this.zoom,
          center: this.center,
          style: style,
          attributionControl: false,
          localFontFamily: '微软雅黑',
          preserveDrawingBuffer: true
        });


        map.on('load', () => {
          this.map = map
          this.$emit('map-loaded', map)
        })
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.map-container {
  width: 100%;
  height: 100%;
  position: relative;

  .map-tools {
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
    z-index: 9;

  }
}
</style>