<script setup lang="ts">
const props = defineProps(['error', 'form', 'valid', 'tainted'])
</script>
<template>
  <div class="form-container">
    <pre>{{ form }}</pre>

    <br />

    <pre>{{ error }}</pre>
  </div>
</template>

<style scoped></style>
