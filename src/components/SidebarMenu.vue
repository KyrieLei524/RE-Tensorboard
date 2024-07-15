<template>
  <div class="sidebar-container">
    <el-menu
      :default-active="defaultActive"
      class="el-menu-vertical-demo"
      @open="handleOpen"
      @close="handleClose"
    >
      <MenuItem v-for="item in menuItems" :key="item.index" :item="item" @item-click="handleItemClick" />
    </el-menu>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import MenuItem from './MenuItem.vue';

const defaultActive = ref('2');

const handleOpen = (key, keyPath) => {
  console.log('open:', key, keyPath);
};

const handleClose = (key, keyPath) => {
  console.log('close:', key, keyPath);
};

const handleItemClick = (path) => {
  console.log('Path:', path);
};

const menuItems = ref([
  {
    index: '1',
    title: 'Navigator One',
    children: [
      {
        index: '1-1',
        title: 'item one',
        children: [
          { index: '1-1-1', title: 'Sub item one' },
          {
            index: '1-1-2',
            title: 'Sub item two',
            children: [
              { index: '1-1-2-1', title: 'Sub sub item one' },
              {
                index: '1-1-2-2',
                title: 'Sub sub item two',
                children: [
                  {
                    index: '1-1-2-2-1',
                    title: 'Sub sub sub item one',
                    children: [
                      { index: '1-1-2-2-1-1', title: 'Sub sub sub sub item one' },
                    ],
                  },
                ],
              },
            ],
          },
        ],
      },
      {
        index: '1-2',
        title: 'item two',
        children: [
          { index: '1-2-1', title: 'Sub item three' },
        ],
      },
    ],
  },
  {
    index: '2',
    title: 'Navigator Two',
  },
  {
    index: '3',
    title: 'Navigator Three',
    disabled: true,
  },
  {
    index: '4',
    title: 'Navigator Four',
  },
]);
</script>

<style scoped>
.sidebar-container {
  width: 250px;
  height: 100vh;
  overflow: hidden;
  position: absolute;
  left:1px;
}

.el-menu-vertical-demo {
  width: 100%;
  height: 100%;
  overflow: auto;
  white-space: nowrap;
}

.el-menu-item,
.el-sub-menu__title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 100%;
}

.el-sub-menu__title {
  display: flex;
  align-items: center;
}

.el-sub-menu__title span {
  flex-grow: 1;
}

.el-menu-item > span {
  flex-grow: 1;
}
</style>
