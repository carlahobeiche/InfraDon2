<script setup lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
 
const counter = ref(0);
const increment = () => {
counter.value++
 
}
const openReadmeInEditor = () => fetch('/__open-in-editor?file=README.md')
</script>
 
<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment"></button>
</template>

