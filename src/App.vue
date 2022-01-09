<template>
  <div id="app">
    <scroll />
  </div>
</template>

<script>
// import groupSelect from './components/group-select.vue'
import scroll from './components/scroll.vue'

export default {
  name: 'App',
  components: {
    // groupSelect,
    scroll
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
