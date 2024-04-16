<script setup>
import './hello.css'
import { ref } from 'vue'

import {page} from './hello'

defineProps({
  msg: String,
})

const pageMsg = ref(page.msg)

</script>

<template>
  <h2 class="red">red: {{ msg }}</h2>
  <h2 >local: {{ pageMsg }}</h2>
  <footer>
    Page Footer
  </footer>
</template>

<style scoped>
@import url('./hello-footer.css');
</style>
