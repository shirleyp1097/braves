<script setup>
import BaseballTable from './components/BaseballTable.vue'
</script>

<template>
  <header>
    <h1>Baseball Statistics Viewer</h1>
  </header>
  <main>
    <BaseballTable />
  </main>
</template>

<style scoped>
header {
  background-color: #1a237e;
  color: white;
  padding: 1rem;
  text-align: center;
}

main {
  padding: 2rem;
}
</style>
