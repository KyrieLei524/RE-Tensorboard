<template>
  <el-sub-menu v-if="hasChildren" :index="item.label">
    <template #title>
      <div class="submenu-title">
        <span @click.stop="handleClick">{{ item.label.toString().split('/').pop() }}</span>
        <el-icon @click.stop="toggleSubmenu">
          <component :is="'el-icon-arrow-down'" />
        </el-icon>
      </div>
    </template>
    <MenuItem
      v-for="child in item.children"
      :key="child.index"
      :item="child"
      @item-click="handleChildClick"
    />
  </el-sub-menu>
  <el-menu-item v-else :index="item.label" @click="handleClick">
    <span>{{ item.label.toString().split('/').pop() }}</span>
  </el-menu-item>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  item: {
    type: Object,
    required: true,
  },
});

const emits = defineEmits(['item-click']);

const hasChildren = props.item.children && props.item.children.length > 0;

const handleClick = () => {
  emits('item-click', props.item.label.toString().split('/').pop());
};

const handleChildClick = (childPath) => {
  emits('item-click', `${props.item.label.toString().split('/').pop()}/${childPath}`);
};

const toggleSubmenu = () => {
  const submenu = document.querySelector(`[aria-controls="submenu${props.item.index}"]`);
  if (submenu) {
    submenu.click();
  }
};
</script>

<style scoped>
.el-sub-menu__title,
.el-menu-item {
  white-space: nowrap;
  text-overflow: ellipsis;
  display: flex;
  align-items: center;
  width: 100%;
}

.el-sub-menu__title span,
.el-menu-item span {
  flex-grow: 1;
}

.submenu-title {
  display: flex;
  align-items: center;
}

.submenu-title .el-icon {
  cursor: pointer;
  margin-left: 10px;
}
</style>
