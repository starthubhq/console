<template>
  <div class="diagram-container">
    <VueFlow
      v-model="elements"
      :default-viewport="{ zoom: 1.5 }"
      :min-zoom="0.2"
      :max-zoom="4"
      class="vue-flow-diagram"
      @node-click="onNodeClick"
      @edge-click="onEdgeClick"
    >
      <Background pattern-color="#aaa" :gap="8" />
    </VueFlow>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import '@vue-flow/core/dist/style.css'
import '@vue-flow/core/dist/theme-default.css'

// Define types for nodes and edges
interface Node {
  id: string
  type: string
  position: { x: number; y: number }
  data: { label: string }
}

interface Edge {
  id: string
  source: string
  target: string
  type: string
}

// Sample data for the placeholder diagram
const elements = ref<(Node | Edge)[]>([
  // Nodes
  {
    id: '1',
    type: 'input',
    position: { x: 250, y: 25 },
    data: { label: 'Start' }
  },
  {
    id: '2',
    type: 'default',
    position: { x: 100, y: 125 },
    data: { label: 'Process A' }
  },
  {
    id: '3',
    type: 'default',
    position: { x: 400, y: 125 },
    data: { label: 'Process B' }
  },
  {
    id: '4',
    type: 'default',
    position: { x: 250, y: 225 },
    data: { label: 'Decision' }
  },
  {
    id: '5',
    type: 'default',
    position: { x: 100, y: 325 },
    data: { label: 'Result A' }
  },
  {
    id: '6',
    type: 'default',
    position: { x: 400, y: 325 },
    data: { label: 'Result B' }
  },
  {
    id: '7',
    type: 'output',
    position: { x: 250, y: 425 },
    data: { label: 'End' }
  },
  
  // Edges
  {
    id: 'e1-2',
    source: '1',
    target: '2',
    type: 'smoothstep'
  },
  {
    id: 'e1-3',
    source: '1',
    target: '3',
    type: 'smoothstep'
  },
  {
    id: 'e2-4',
    source: '2',
    target: '4',
    type: 'smoothstep'
  },
  {
    id: 'e3-4',
    source: '3',
    target: '4',
    type: 'smoothstep'
  },
  {
    id: 'e4-5',
    source: '4',
    target: '5',
    type: 'smoothstep'
  },
  {
    id: 'e4-6',
    source: '4',
    target: '6',
    type: 'smoothstep'
  },
  {
    id: 'e5-7',
    source: '5',
    target: '7',
    type: 'smoothstep'
  },
  {
    id: 'e6-7',
    source: '6',
    target: '7',
    type: 'smoothstep'
  }
])

// Event handlers
const onNodeClick = (event: any) => {
  console.log('Node clicked:', event.node)
}

const onEdgeClick = (event: any) => {
  console.log('Edge clicked:', event.edge)
}
</script>

<style scoped>
.diagram-container {
  width: 100%;
  height: 100%;
  min-height: 500px;
}

.vue-flow-diagram {
  background-color: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 8px;
}

:deep(.vue-flow__node) {
  padding: 10px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

:deep(.vue-flow__node-input) {
  background: #e3f2fd;
  border: 2px solid #2196f3;
  color: #1976d2;
}

:deep(.vue-flow__node-output) {
  background: #f3e5f5;
  border: 2px solid #9c27b0;
  color: #7b1fa2;
}

:deep(.vue-flow__node-default) {
  background: #f1f8e9;
  border: 2px solid #4caf50;
  color: #388e3c;
}

:deep(.vue-flow__edge-path) {
  stroke: #666;
  stroke-width: 2;
}
</style>
