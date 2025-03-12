<script setup lang="ts">
import MapComponent from '@/components/Map.vue';
</script>

<template>
  <main>
    <div class="hero">
      <img src="/images/alta-brand.svg" width="100">
      <h2>Portland Bike Facilities Map</h2>
      <cite>by Jeremy Wilson</cite>
    </div>
    <MapComponent />
  </main>
</template>

<style scoped>
.hero {
  position:absolute;
  top:20px;
  left:20px;
  background:rgba(255,255,255,0.75);
  z-index:10;
  padding:20px;
  line-height:1;
  border-radius:3px;
  box-shadow:0px 0px 10px rgba(0,0,0,0.33);
  img { 
    padding-bottom:14px; 
  }
  h2 { 
    font-family:"Raleway", sans-serif; 
    font-weight:700;
    font-size:24px;
    color:var(--color-blue);
    padding-bottom:14px;
  }
  cite {
    font-family:"Open Sans", sans-serif;
    font-size:12px;
    font-style:normal;
    font-weight:700;
    color:var(--color-navy);
  }
}
</style>