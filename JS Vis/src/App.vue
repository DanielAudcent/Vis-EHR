<template>
<!-- html code goes here -->
  <div id="app">
    <Shapefile>
    </Shapefile>
  </div>
</template>

<script>
// js code goes here
import Shapefile from './components/Shapefile.vue'
 
export default {
  name: 'App',
  components: {
    Shapefile
  }
}
</script>

<style>
/* css code goes here */
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
