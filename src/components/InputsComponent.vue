<script setup lang="ts">
import { onMounted, watch } from 'vue'
import { useRoute } from 'vue-router';

const route = useRoute();

onMounted(() => {
  const namespace = String(route.params.namespace)
  const slug = String(route.params.slug)
  console.info(namespace)
  console.info(slug)
  // if (namespace && packageName) initInputs(namespace, packageName)
})

// 2️⃣ React if the URL changes or if the param shows up later
watch(
  () => route.query.package,
  (pkg) => {
    const namespace = String(route.params.namespace)
    const slug = String(route.params.slug)
    console.info(namespace)
    console.info(slug)
    // if (typeof pkg === 'string' && pkg.trim().length) initInputs(namespace, packageName)
  }
)

</script>

<template>
  inputs
</template>

<style scoped>

</style>
