<script setup>
import temp from './components/tt.vue';
import sidebar from './components/SidebarMenu.vue';
</script>

<template>

  <temp></temp>

</template>

<style scoped>
.container {
  display: flex;
}

.left-panel {
  flex: 1;
  /* 左侧面板占据剩余空间 */
}

.right-panel {
  flex: 1;
  /* 右侧面板占据剩余空间 */
}
</style>
