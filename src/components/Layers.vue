<template>
  <div id="control">
    <ul>
      <li
        v-bind:item="layer"
        v-bind:index="index"
        v-bind:key="layer.Name"
        v-for="(layer, index) in layers"
      >
        <input type="checkbox" v-model="layer.active" v-on:click="drawLayer(layer)" />
        <span v-if="layer.Name === 'gis_example'">Все слои</span>
        <span v-else>{{layer.Name}}</span>
      </li>
    </ul>
  </div>
</template>

<style lang="scss">
#control {
  background-color: rgba(255, 255, 255, 0.8);
  z-index: 1;
  position: absolute;
  right: 10%;
  bottom: 50%;
}
ul {
  margin: 0;
  padding: 0;
}
ul li {
  list-style-type: none;
}
</style>

<script>
import WMSCapabilities from "ol/format/WMSCapabilities";
import axios from "axios";
import { eventBus } from "../main.js";
export default {
  data() {
    return {
      layers: []
    };
  },
  mounted: function() {
    this.getLayers();
  },

  created() {
    eventBus.$on("layer-updated", layerName => {
      let index = this.layers.findIndex(e => e.Name === layerName);
      this.layers[index].active = true;
      this.layers = this.layers.slice();
      let targetLayers = this.layers.filter(e => e.active);
      this.submitLayers(targetLayers);
    });
  },

  methods: {
    drawLayer(layer) {
      layer.active = !layer.active;
      let targetLayers = [];
      if (layer.Name === "gis_example") {
        if (layer.active) {
          this.layers.forEach(e => (e.active = true));
          targetLayers = this.layers.filter(e => e.Name !== "gis_example");
        } else {
          this.layers.forEach(e => (e.active = false));
        }
        this.layers = this.layers.slice();
      } else {
        targetLayers = this.layers.filter(e => e.active);
      }
      this.submitLayers(targetLayers);
    },

    submitLayers(layers) {
      console.log("drawLayer", layers);
      eventBus.$emit("layers-submitted", layers);
    },

    drawLayers(layers) {
      console.log("drawLayers", layers);
      eventBus.$emit("layers-submitted", layers);
    },

    getLayers() {
      const formatter = new WMSCapabilities();
      const endpoint =
        "http://localhost:8080/geoserver/gis_example/wms?service=wms&request=GetCapabilities";
      axios.get(endpoint).then(response => {
        console.log(response);

        this.layers = formatter.read(response.data).Capability.Layer.Layer;

        console.log(this.layers);

        this.drawLayers(this.layers);
      });
    }
  }
};
</script>
