<template>
  <div class="tabs">
    <div class="tab-headers">
      <button v-for="(tab, index) in tabs" :key="index" @click="activateTab(index)" :class="{ active: activeTab === index }">{{ tab.label }}</button>
    </div>
    <div class="tab-content">
      <slot></slot>
    </div>
  </div>
</template>

<script>
import { defineComponent, ref, provide } from 'vue';

export default defineComponent({
  name: 'Tabs',
  props: {
    activeTab: {
      type: Number,
      default: 0
    }
  },
  setup(props) {
    const activeTab = ref(props.activeTab);
    const tabs = ref([]);

    const registerTab = (tab) => {
      tabs.value.push(tab);
      tab.activateTab = activateTab; // 将激活选项卡的方法传递给 Tab 组件
    };

    const activateTab = (index) => {
      activeTab.value = index;
      // 同步显示隐藏
      tabs.value.forEach((tab, i) => {
        tab.isActive.value = index === i;
      });
    };

    provide('tabs', tabs);

    return {
      activeTab,
      registerTab
    };
  }
});
</script>

<style>
.tab-headers {
  display: flex;
}

.tab-headers button {
  padding: 10px;
  cursor: pointer;
  border: none;
  background-color: transparent;
}

.tab-headers button.active {
  font-weight: bold;
}

.tab-content {
  margin-top: 10px;
}
</style>
