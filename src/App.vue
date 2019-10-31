<template>
  <div id="app">
    <Map />
    <Layers />
  </div>
</template>

<script>
import Map from "./components/Map.vue";
import Layers from "./components/Layers.vue";

export default {
  name: "app",
  components: {
    Map,
    Layers
  }
};
</script>

<style lang="scss">
#app,
html,
body {
  margin: 0;
  width: 100%;
  height: 100%;
}
::-webkit-scrollbar {
  display: none;
}
</style>
