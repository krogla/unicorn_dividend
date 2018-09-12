<template>
  <div id="app">
    <img alt="UnicornGO" src="./assets/logo.png">
    <MyDividends/>
  </div>
</template>

<script>
  import MyDividends from './components/MyDividends.vue'

  export default {
    name: 'app',
    components: {
      MyDividends
    }
  }
</script>

<style>
  #app {
    /*font-family: 'Avenir', Helvetica, Arial, sans-serif;*/
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    /*color: #2c3e50;*/
    /*margin-top: 60px;*/
  }
</style>
