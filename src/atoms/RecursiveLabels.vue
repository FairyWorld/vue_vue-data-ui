<template>
    <template v-for="node in dataset">
        <template v-if="node.polygonPath && node.polygonPath.coordinates">
            <template v-for="(coordinate) in node.polygonPath.coordinates">
                <text
                    data-cy="recursive-label"
                    :x="coordinate.x" 
                    :y="coordinate.y + node.circleRadius *2" 
                    :fill="color"
                    :font-size="node.circleRadius"
                    text-anchor="middle"
                    style="opacity:0.8; pointer-events: none;"
                >
                    {{ node.name }}
                </text>
            </template>
            <template v-if="node.nodes && node.nodes.length > 0">
                <RecursiveLabels 
                    :dataset="node.nodes" 
                    :color="color" 
                    :hoveredUid="hoveredUid"
                />
            </template>
        </template>
    </template>
</template>

<script setup>
import { toRef } from 'vue';

const emit = defineEmits(["zoom", 'hover']);

const props = defineProps({
    color: {
        type: String,
        default: '#000000',
    },
    dataset: {
        type: Array,
        default: () => [],
    },
    hoveredUid: {
        type: String,
        default: null
    },
});

const nodes = toRef(props, 'dataset');

nodes.value.forEach((node) => {
    if (node.nodes && node.nodes.length > 0) {
        node.nodes.forEach((childNode) => {
            childNode.ancestor = node;
        });
    }
});
</script>
