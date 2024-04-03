<template>
  <div>
    <CallHealing119Button v-if="isRerender" @on-rerender="handleReRender" />
  </div>
</template>

<script setup lang="ts">
import CallHealing119Button from './CallHealing119Button.vue'

const isRerender = ref(true)
const handleReRender = (val: boolean) => {
  isRerender.value = val
  nextTick(() => {
    isRerender.value = true
  })
}
</script>
