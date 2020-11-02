<template>
<div id="app">
  <div id="map"></div>
  <Panel />
</div>
</template>

<script>
import Panel from "./components/Panel.vue";
import "./assets/custom.scss";
export default {
  name: "app",
  components: {
    Panel
  }
};
</script>
