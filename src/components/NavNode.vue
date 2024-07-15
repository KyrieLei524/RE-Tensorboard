<template>
    <div class="nav-node">
      <div @click="toggleExpand" class="nav-node-label">
        {{ node.label.split('/').pop() }}
        <button v-if="node.children.length > 0" @click.stop="toggleExpand">
          {{ node.expanded ? '-' : '+' }}
        </button>
      </div>
      <div v-if="node.expanded" class="nav-node-children">
        <nav-node v-for="child in node.children" :key="child.label" :node="child" />
      </div>
    </div>
  </template>
  
  <script setup>
  import { reactive, defineProps } from 'vue';
  
  const props = defineProps({
    node: {
      type: Object,
      required: true,
    },
  });
  
  const state = reactive({
    expanded: props.node.expanded,
  });
  
  const toggleExpand = () => {
    props.node.expanded = !props.node.expanded;
    state.expanded = props.node.expanded;
  };
  </script>
  
  <style scoped>
  .nav-node {
    margin-left: 10px;
  }
  
  .nav-node-label {
    cursor: pointer;
    padding: 5px;
  }
  
  .nav-node-label button {
    margin-left: 10px;
  }
  
  .nav-node-children {
    padding-left: 20px;
  }
  </style>