<template>
  <div >
    <div v-show="isActive">
      <slot ></slot>
    </div>
  </div>
</template>
<script>
import { defineComponent, ref, inject, onMounted } from 'vue';

export default defineComponent({
  name: 'Tab',
  props: {
    label: {
      type: String,
      required: true
    }
  },
  setup(props) {
    const isActive = ref(false);
    const tabs = inject('tabs');
    let activateTab;

    onMounted(() => {
      tabs.registerTab({ label: props.label, isActive });
      activateTab = tabs.activateTab;
    });

    const handleClick = () => {
      activateTab(tabs.value.indexOf(isActive));
    };

    return {
      isActive,
      handleClick
    };
  }
});
</script>

<style>
.tab-panel {
  display: none;
}

.tab-panel.active {
  display: block;
}
</style>
