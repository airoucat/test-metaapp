<script setup lang="ts">
import { ref } from 'vue'

defineProps<{ msg: string }>()

const count = ref(0)
</script>

<template>
  <div class="wrapper">

  </div>
</template>

<style scoped lang="scss">
.wrapper {
  background-color: #b6b0b0;
  width: 150px;
  // border: 1px solid #7b7979;
  height: 200px;
}
</style>
