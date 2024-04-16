<script setup>
import { reactive } from 'vue';

const obj = reactive({
    msg: 'async component'
})
</script>

<template>
  <h1 class="async">{{ obj.msg }}</h1>
</template>

<style scoped>
@import url('./async.css');
</style>
