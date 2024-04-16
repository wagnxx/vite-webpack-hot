

<script setup>
import { defineAsyncComponent,ref } from 'vue';

const AsyncComponent = defineAsyncComponent(() => import('./components/AsyncComponent.vue'));
import HelloWorld from './components/hello-world/HelloWorld.vue'
const idx = ref(0)
const clickTab = i => {
  idx.value = i;
}

const activeTab = ref(1);
</script>

<template>
  <div>
    <div class="tabs">
      <ul>
        <li @click="clickTab(0)">Hello</li>
        <li @click="clickTab(1)">async</li>
      </ul>
      <div class="content">
        <HelloWorld v-if="idx === 0" msg="Hello world1"/>
        <AsyncComponent v-if="idx === 1"/>
      </div>
    </div>

    <!-- <HelloWorld msg="Vite - Vue" /> -->
  </div>
</template>

<style scoped>
.tabs{

}
.tabs ul {
  list-style: none;
  display: flex;
}
.tabs ul  li {
  border: 1px solid #ddd;
  padding: 4px 8px;
}
</style>
