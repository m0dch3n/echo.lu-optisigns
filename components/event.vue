<script setup lang="ts">
const props = defineProps({
  event: {
    type: Object,
    required: true
  }
})
</script>

<template>
  <div>
    {{ props.event?.title }}
  </div>
</template>
