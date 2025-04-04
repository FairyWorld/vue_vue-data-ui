<script setup>
import { ref, computed } from "vue";
import LocalVueUiSparkStackbar from '../src/components/vue-ui-sparkstackbar.vue';
import LocalVueDataUi from '../src/components/vue-data-ui.vue';
import Box from "./Box.vue";
import convertArrayToObject from "./convertModel";

const dataset = ref(
    [
        {
            name: "Vue",
            value: 258,
        },
        {
            name: "Javascript",
            value: 36,
        },
        {
            name: "Other",
            value: 16,
        },
    ]
);

const alternateDataset = ref([
        {
            name: "ALT 1",
            value: 50,
        },
        {
            name: "ALT 2",
            value: 50,
        },
])

const alternateConfig = ref({
    style: {
        backgroundColor: '#CCCCCC',
        title: {
            text: 'Alternate'
        }
    }
})

const isPropsToggled = ref(false);
function toggleProps() {
    isPropsToggled.value = !isPropsToggled.value;
}

function alterDataset() {
    dataset.value.push({
        name: 'Added',
        value: Math.round(Math.random() * 30)
    })
}

const model = ref([
    { key: 'style.backgroundColor', def: '#FFFFFF20', type: 'color'},
    { key: 'style.fontFamily', def: 'inherit', type: 'text'},
    { key: 'style.animation.show', def: true, type: 'checkbox'},
    { key: 'style.animation.animationFrames', def: 60, type: 'number', min: 0, max: 300},
    { key: 'style.bar.gradient.show', def: true, type: 'checkbox'},
    { key: 'style.bar.gradient.intensity', def: 40, type: 'range', min: 0, max: 100},
    { key: 'style.bar.gradient.underlayerColor', def: '#FFFFFF20', type: 'color'},
    { key: 'style.legend.textAlign', def: 'left', type: 'select', options: ['left', 'center', 'right']},
    { key: 'style.legend.show', def: true, type: 'checkbox'},
    { key: 'style.legend.fontSize', def: 12, type: 'number', min: 8, max: 48},
    { key: 'style.legend.margin', def: '0 0 6px 0', type: 'text'},
    { key: 'style.legend.name.color', def: '#1A1A1A', type: 'color'},
    { key: 'style.legend.name.bold', def: false, type: 'checkbox'},
    { key: 'style.legend.value.show', def: true, type: 'checkbox'},
    { key: 'style.legend.value.bold', def: false, type: 'checkbox'},
    { key: 'style.legend.value.color', def: '#1A1A1A', type: 'color'},
    { key: 'style.legend.value.prefix', def: 'P', type: 'text'},
    { key: 'style.legend.value.suffix', def: 'S', type: 'text'},
    { key: 'style.legend.value.rounding', def: 0, type: 'number', min: 0, max: 12},
    { key: 'style.legend.percentage.show', def: true, type: 'checkbox'},
    { key: 'style.legend.percentage.bold', def: true, type: 'checkbox'},
    { key: 'style.legend.percentage.color', def: '#1A1A1A', type: 'color'},
    { key: 'style.legend.percentage.rounding', def: 2, type: 'number', min: 0, max: 12},
    { key: 'style.title.textAlign', def: 'left', type: 'select', options: ['left', 'center', 'right']},
    { key: 'style.title.text', def: 'Lorem ipsum dolor sic amet', type: 'text'},
    { key: 'style.title.color', def: '#1A1A1A', type: 'color'},
    { key: 'style.title.fontSize', def: 16, type: 'number', min: 8, max: 48},
    { key: 'style.title.bold', def: true, type: 'checkbox'},
    { key: 'style.title.margin', def: '0 0 6px 0', type: 'text'},
    { key: 'style.title.subtitle.color', def: '#A1A1A1', type: 'color'},
    { key: 'style.title.subtitle.text', def: 'Lorem ipsum dolor sic amet'},
    { key: 'style.title.subtitle.fontSize', def: 12, type: 'number', min:8, max: 24},
    { key: 'style.title.subtitle.bold', def: false, type: 'checkbox'},

    { key: 'style.tooltip.show', def: true, type: 'checkbox', label: 'show', category: 'tooltip' },
    { key: 'style.tooltip.backgroundColor', def: '#FFFFFF', type: 'color', label: 'backgroundColor', category: 'tooltip' },
    { key: 'style.tooltip.color', def: '#1A1A1A', type: 'color', label: 'textColor', category: 'tooltip' },
    { key: 'style.tooltip.fontSize', def: 14, type: 'number', min: 6, max: 24, label: 'fontSize', category: 'tooltip' },
    { key: 'style.tooltip.backgroundOpacity', def: 60, type: 'range', min: 0, max: 100},
    { key: 'style.tooltip.position', def: 'center', type: 'select', options: ['left', 'center', 'right']},
    { key: 'style.tooltip.offsetY', def: 24, type: 'number', min: 0, max: 48},
])

const themeOptions = ref([
    "",
    "hack",
    "zen",
    "concrete",
    "default",
    "celebration",
    "celebrationNight"
])

const currentTheme = ref(themeOptions.value[6])

const config = computed(() => {
    const c = convertArrayToObject(model.value);
    return {
        ...c,
        style: {
            ...c.style,
            legend: {
                ...c.style.legend,
                value: {
                    ...c.style.legend.value,
                    formatter: ({value, config }) => {
                        // console.log(config)
                        return `f - ${value}`
                    }
                }
            },
            tooltip: {
                ...c.style.tooltip,
                // customFormat: ({ datapoint }) => {
                //     return datapoint.name
                // }
            }
        },
        theme: currentTheme.value,
        customPalette: ['#6376DD', "#DD3322", "#66DDAA"],
    }
});

const step = ref(0)

</script>

<template>
    <div style="margin: 12px 0; color: white">
        Theme:
        <select v-model="currentTheme" @change="step += 1">
            <option v-for="opt in themeOptions">{{ opt }}</option>
        </select>
    </div>

    <button @click="toggleProps">TOGGLE PROPS: {{ isPropsToggled }}</button>
    <button @click="alterDataset">ALTER DATASET</button>

    <Box comp="VueUiSparkStackbar" :dataset="dataset">
        <template #title>VueUiSparkStackbar</template>

        <template #local>
            <LocalVueUiSparkStackbar :dataset="isPropsToggled ? alternateDataset : dataset" :config="isPropsToggled ? alternateConfig : config" :key="`local_${step}`">
                <template #chart-background>
                    <div style="width: 100%; height: 100%; background: radial-gradient(at top left, red, white)"/>
                </template>

                <template #source>
                    <div style="width:100%;font-size:10px;text-align:left">
                        SOURCE: Lorem ipsum dolor sit, amet consectetur adipisicing elit. Tenetur, molestiae perspiciatis nam quae libero, deserunt in aperiam unde officia sint saepe laboriosam ducimus aspernatur labore! Sapiente aspernatur corrupti quis ad.
                    </div>
                </template>
            </LocalVueUiSparkStackbar>
        </template>

        <template #VDUI-local>
            <LocalVueDataUi component="VueUiSparkStackbar" :dataset="isPropsToggled ? alternateDataset : dataset" :config="isPropsToggled ? alternateConfig : config" :key="`VDUI-lodal_${step}`">
            </LocalVueDataUi>
        </template>

        <template #build>
            <VueUiSparkStackbar :dataset="isPropsToggled ? alternateDataset : dataset" :config="isPropsToggled ? alternateConfig : config" :key="`build_${step}`">
            </VueUiSparkStackbar>
        </template>

        <template #VDUI-build>
            <VueDataUi component="VueUiSparkStackbar" :dataset="isPropsToggled ? alternateDataset : dataset" :config="isPropsToggled ? alternateConfig : config" :key="`VDUI-build_${step}`">
            </VueDataUi>
        </template>

        <template #knobs>
            <div
                style="display: flex; flex-direction: row; flex-wrap:wrap; align-items:center; width: 100%; color: #CCCCCC; gap:24px;">
                <div v-for="knob in model">
                    <label style="font-size: 10px">{{ knob.key }}</label>
                    <div
                        style="display:flex; flex-direction:row; flex-wrap: wrap; align-items:center; gap:6px; height: 40px">
                        <input v-if="!['none', 'select'].includes(knob.type)" :step="knob.step" :type="knob.type" :min="knob.min ?? 0"
                            :max="knob.max ?? 0" v-model="knob.def" @change="step += 1">
                        <select v-if="knob.type === 'select'" v-model="knob.def" @change="step += 1">
                            <option v-for="opt in knob.options">{{ opt }}</option>
                        </select>
                    </div>
                </div>
            </div>
        </template>

        <template #config>
            {{ config }}
        </template>
    </Box>
</template>