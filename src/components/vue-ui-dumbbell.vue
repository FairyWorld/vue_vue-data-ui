<script setup>
import { ref, computed, nextTick, onMounted, onBeforeUnmount, watch, defineAsyncComponent, shallowRef } from "vue";
import {
    applyDataLabel,
    calculateNiceScale,
    checkNaN,
    createCsvContent,
    createUid,
    darkenHexColor,
    dataLabel,
    downloadCsv,
    error,
    getMissingDatasetAttributes,
    lightenHexColor,
    objectIsEmpty,
    XMLNS,
} from "../lib";
import { throttle } from "../canvas-lib";
import { useNestedProp } from "../useNestedProp";
import { usePrinter } from "../usePrinter";
import { useResponsive } from "../useResponsive";
import { useConfig } from "../useConfig";
import { useUserOptionState } from "../useUserOptionState";
import { useChartAccessibility } from "../useChartAccessibility";
import themes from "../themes.json";
import Legend from "../atoms/Legend.vue"; // Must be ready in responsive mode
import Title from "../atoms/Title.vue"; // Must be ready in responsive mode
import img from "../img";

const Accordion = defineAsyncComponent(() => import('./vue-ui-accordion.vue'));
const DataTable = defineAsyncComponent(() => import('../atoms/DataTable.vue'));
const PackageVersion = defineAsyncComponent(() => import('../atoms/PackageVersion.vue'));
const PenAndPaper = defineAsyncComponent(() => import('../atoms/PenAndPaper.vue'));
const Skeleton = defineAsyncComponent(() => import('./vue-ui-skeleton.vue'));
const UserOptions = defineAsyncComponent(() => import('../atoms/UserOptions.vue'));

const { vue_ui_dumbbell: DEFAULT_CONFIG } = useConfig();

const props = defineProps({
    config: {
        type: Object,
        default() {
            return {}
        }
    },
    dataset: {
        type: Array,
        default() {
            return []
        }
    }
});

const emit = defineEmits(['selectDatapoint']);

const isDataset = computed({
    get() {
        return !!props.dataset && props.dataset.length;
    },
    set(bool){
        return bool;
    }
});

const uid = ref(createUid());
const step = ref(0);
const dumbbellChart = ref(null);
const chartTitle = ref(null);
const chartLegend = ref(null);
const source = ref(null);
const noTitle = ref(null);
const titleStep = ref(0);
const tableStep = ref(0);
const legendStep = ref(0);

const FINAL_CONFIG = computed({
    get: () => {
        return prepareConfig();
    },
    set: (newCfg) => {
        return newCfg
    }
});

const { userOptionsVisible, setUserOptionsVisibility, keepUserOptionState } = useUserOptionState({ config: FINAL_CONFIG.value });
const { svgRef } = useChartAccessibility({ config: FINAL_CONFIG.value.style.chart.title });

function prepareConfig() {
    const mergedConfig = useNestedProp({
        userConfig: props.config,
        defaultConfig: DEFAULT_CONFIG
    });
    if (mergedConfig.theme) {
        return {
            ...useNestedProp({
                userConfig: themes.vue_ui_dumbbell[mergedConfig.theme] || props.config,
                defaultConfig: mergedConfig
            }),
        }
    } else {
        return mergedConfig;
    }
}

watch(() => props.config, (_newCfg) => {
    FINAL_CONFIG.value = prepareConfig();
    userOptionsVisible.value = !FINAL_CONFIG.value.userOptions.showOnChartHover;
    prepareChart();
    titleStep.value += 1;
    tableStep.value += 1;
    legendStep.value += 1;
    baseRowHeight.value = FINAL_CONFIG.value.style.chart.rowHeight;
    baseWidth.value = FINAL_CONFIG.value.style.chart.width;

    // Reset mutable config
    mutableConfig.value.showTable = FINAL_CONFIG.value.table.show;
}, { deep: true });

watch(() => props.dataset, (_) => {
    prepareDataset();
}, { deep: true })

const resizeObserver = shallowRef(null);
const observedEl = shallowRef(null);

onMounted(() => {
    prepareChart();
});

function prepareChart() {
    if(objectIsEmpty(props.dataset)) {
        error({
            componentName: 'VueUiDumbbell',
            type: 'dataset'
        })
    } else {
        props.dataset.forEach((ds, i) => {
            getMissingDatasetAttributes({
                datasetObject: ds,
                requiredAttributes: ['name', 'start', 'end']
            }).forEach(attr => {
                isDataset.value = false;
                error({
                    componentName: 'VueUiDumbbell',
                    type: 'datasetSerieAttribute',
                    property: attr,
                    index: i
                })
            })
        });
    }

    if (FINAL_CONFIG.value.responsive) {
        const handleResize = throttle(() => {
            const { width, height } = useResponsive({
                chart: dumbbellChart.value,
                title: FINAL_CONFIG.value.style.chart.title.text ? chartTitle.value : null,
                legend: FINAL_CONFIG.value.style.chart.legend.show ? chartLegend.value : null,
                source: source.value,
                noTitle: noTitle.value
            });

            requestAnimationFrame(() => {
                baseWidth.value = width;
                baseRowHeight.value = height / props.dataset.length;
                mutableDataset.value = getMutableDataset();
            });
        });

        if (resizeObserver.value) {
            if (observedEl.value) {
                resizeObserver.value.unobserve(observedEl.value);
            }
            resizeObserver.value.disconnect();
        }

        resizeObserver.value = new ResizeObserver(handleResize);
        observedEl.value = dumbbellChart.value.parentNode;
        resizeObserver.value.observe(observedEl.value);
    }
}

onBeforeUnmount(() => {
    if (resizeObserver.value) {
        if (observedEl.value) {
            resizeObserver.value.unobserve(observedEl.value);
        }
        resizeObserver.value.disconnect();
    }
});

const { isPrinting, isImaging, generatePdf, generateImage } = usePrinter({
    elementId: `dumbbell_${uid.value}`,
    fileName: FINAL_CONFIG.value.style.chart.title.text || 'vue-ui-dumbbell',
    options: FINAL_CONFIG.value.userOptions.print
});

const hasOptionsNoTitle = computed(() => {
    return FINAL_CONFIG.value.userOptions.show && !FINAL_CONFIG.value.style.chart.title.text;
});

const mutableConfig = ref({
    showTable: FINAL_CONFIG.value.table.show,
});

const immutableDataset = computed(() => {
    return props.dataset.map((ds, i) => {
        return {
            ...ds,
            start: checkNaN(ds.start),
            end: checkNaN(ds.end),
            id: createUid()
        }
    })
});

const extremes = computed(() => {
    return {
        max: Math.max(...immutableDataset.value.flatMap(ds => [ds.start, ds.end])),
        min: Math.min(...immutableDataset.value.flatMap(ds => [ds.start, ds.end]))
    }
});

const scale = computed(() => {
    return calculateNiceScale(extremes.value.min < 0 ? extremes.value.min : 0, extremes.value.max, FINAL_CONFIG.value.style.chart.grid.scaleSteps)
});

const baseRowHeight = ref(FINAL_CONFIG.value.style.chart.rowHeight);
const baseWidth = ref(FINAL_CONFIG.value.style.chart.width)

const drawingArea = computed(() => {
    const rowHeight = baseRowHeight.value;
    const absoluteWidth = FINAL_CONFIG.value.style.chart.padding.left + FINAL_CONFIG.value.style.chart.padding.right + baseWidth.value;
    const absoluteHeight = FINAL_CONFIG.value.style.chart.padding.top + FINAL_CONFIG.value.style.chart.padding.bottom + rowHeight * props.dataset.length;
    const widthPlotReference = (scale.value.ticks.length) * (baseWidth.value / scale.value.ticks.length)

    return {
        left: FINAL_CONFIG.value.style.chart.padding.left,
        right: absoluteWidth - FINAL_CONFIG.value.style.chart.padding.right,
        top: FINAL_CONFIG.value.style.chart.padding.top,
        bottom: absoluteHeight - FINAL_CONFIG.value.style.chart.padding.bottom,
        width: baseWidth.value,
        height: rowHeight * props.dataset.length,
        rowHeight,
        absoluteHeight,
        absoluteWidth,
        widthPlotReference
    }
});

function getMutableDataset() {
    return immutableDataset.value.map((ds, i) => {
        const startX = drawingArea.value.left + (((ds.start + Math.abs(scale.value.min)) / (scale.value.max + Math.abs(scale.value.min))) * drawingArea.value.widthPlotReference);
        const endX = drawingArea.value.left + (((ds.end + Math.abs(scale.value.min)) / (scale.value.max + Math.abs(scale.value.min))) * drawingArea.value.widthPlotReference);
        const centerX = startX + ((endX - startX) / 2)
        return {
            ...ds,
            startX,
            endX,
            centerX,
            y: drawingArea.value.top + (i * baseRowHeight.value) + (baseRowHeight.value / 2),
            endVal: ds.start
        }
    })
}

const mutableDataset = ref([])

const raf = ref(null);
const grandTotalEnd = computed(() => {
    return immutableDataset.value.map(ds => ds.end).reduce((a, b) => a + b, 0);
});

onMounted(() => {
    prepareDataset();
})

function prepareDataset() {
    mutableDataset.value = getMutableDataset();

    let totalEnd = mutableDataset.value.map(ds => ds.start).reduce((a, b) => a + b, 0);

    function anim() {
        const diffs = immutableDataset.value.map(ds => {
            return ds.end - ds.start
        })

        if(totalEnd >= grandTotalEnd.value) {
            cancelAnimationFrame(raf.value);
            mutableDataset.value = getMutableDataset();
        } else {
            mutableDataset.value = mutableDataset.value.map((ds, i) => {
                ds.endVal += diffs[i] * (FINAL_CONFIG.value.animationSpeed / 100);
                const startX = drawingArea.value.left + (((ds.start + Math.abs(scale.value.min)) / (scale.value.max + Math.abs(scale.value.min))) * drawingArea.value.widthPlotReference);
                const endX = drawingArea.value.left + (((ds.endVal + Math.abs(scale.value.min)) / (scale.value.max + Math.abs(scale.value.min))) * drawingArea.value.widthPlotReference);
                const centerX = startX + ((endX - startX) / 2)
                return {
                    ...ds,
                    startX,
                    endX,
                    centerX,
                    y: drawingArea.value.top + (i * baseRowHeight.value) + (baseRowHeight.value / 2),
                    endVal: ds.endVal
                }
            })
            totalEnd = mutableDataset.value.map(ds => ds.endVal).reduce((a, b) => a + b, 0);
            raf.value = requestAnimationFrame(anim)
        }
    }
    if(FINAL_CONFIG.value.useAnimation) {
        anim()
    } else {
        mutableDataset.value = getMutableDataset()
    }
}

const legendSet = computed(() => {
    return [
        { name: FINAL_CONFIG.value.style.chart.legend.labelStart, color: FINAL_CONFIG.value.style.chart.plots.gradient.show ? `url(#start_grad_${uid.value})` : FINAL_CONFIG.value.style.chart.plots.startColor},
        { name: FINAL_CONFIG.value.style.chart.legend.labelEnd, color: FINAL_CONFIG.value.style.chart.plots.gradient.show ? `url(#end_grad_${uid.value})` : FINAL_CONFIG.value.style.chart.plots.endColor},
    ]
})

const legendConfig = computed(() => {
    return {
        cy: 'donut-div-legend',
        backgroundColor: FINAL_CONFIG.value.style.chart.legend.backgroundColor,
        color: FINAL_CONFIG.value.style.chart.legend.color,
        fontSize: FINAL_CONFIG.value.style.chart.legend.fontSize,
        paddingBottom: 12,
        paddingTop: 12,
        fontWeight: FINAL_CONFIG.value.style.chart.legend.bold ? 'bold' : ''
    }
});

const table = computed(() => {
    const head = mutableDataset.value.map(ds => {
        return {
            name: ds.name,
        }
    });
    const body = mutableDataset.value.map(ds => {
        return {
            start: ds.start,
            end: ds.end
        }
    });
    return { head, body }
});

const dataTable = computed(() => {
    const head = [
        FINAL_CONFIG.value.table.columnNames.series,
        FINAL_CONFIG.value.table.columnNames.start,
        FINAL_CONFIG.value.table.columnNames.end,
        FINAL_CONFIG.value.table.columnNames.progression
    ];
    const body = table.value.head.map((h,i) => {
        const labelStart = dataLabel({
            p: FINAL_CONFIG.value.style.chart.labels.prefix,
            v: table.value.body[i].start,
            s: FINAL_CONFIG.value.style.chart.labels.suffix,
            r: FINAL_CONFIG.value.table.td.roundingValue
        });
        const labelEnd = dataLabel({
            p: FINAL_CONFIG.value.style.chart.labels.prefix,
            v: table.value.body[i].end,
            s: FINAL_CONFIG.value.style.chart.labels.suffix,
            r: FINAL_CONFIG.value.table.td.roundingValue
        });
        const labelProgression = dataLabel({
            v: 100 * ((table.value.body[i].end / table.value.body[i].start) - 1),
            s: '%',
            r: FINAL_CONFIG.value.table.td.roundingPercentage
        })
        return [
            { name: h.name },
            labelStart,
            labelEnd,
            labelProgression
        ]
    });

    const config = {
        th: {
            backgroundColor: FINAL_CONFIG.value.table.th.backgroundColor,
            color: FINAL_CONFIG.value.table.th.color,
            outline: FINAL_CONFIG.value.table.th.outline
        },
        td: {
            backgroundColor: FINAL_CONFIG.value.table.td.backgroundColor,
            color: FINAL_CONFIG.value.table.td.color,
            outline: FINAL_CONFIG.value.table.td.outline
        },
        breakpoint: FINAL_CONFIG.value.table.responsiveBreakpoint
    }

    const colNames = [
        FINAL_CONFIG.value.table.columnNames.series,
        FINAL_CONFIG.value.table.columnNames.start,
        FINAL_CONFIG.value.table.columnNames.end,
        FINAL_CONFIG.value.table.columnNames.progression
    ];

    return {
        colNames,
        head,
        body,
        config
    }
});

function generateCsv(callback=null) {
    nextTick(() => {
        const labels = table.value.head.map((h,i) => {
            return [[
                h.name
            ],[table.value.body[i].start],[table.value.body[i].end]]
        });
        const tableXls = [[FINAL_CONFIG.value.style.chart.title.text],[FINAL_CONFIG.value.style.chart.title.subtitle.text],[[FINAL_CONFIG.value.table.columnNames.series],[FINAL_CONFIG.value.table.columnNames.start],[FINAL_CONFIG.value.table.columnNames.end]]].concat(labels);

        const csvContent = createCsvContent(tableXls);

        if (!callback) {
            downloadCsv({ csvContent, title: FINAL_CONFIG.value.style.chart.title.text || "vue-ui-dumbbell" })
        } else {
            callback(csvContent);
        }
    });
}

const isFullscreen = ref(false)
function toggleFullscreen(state) {
    isFullscreen.value = state;
    step.value += 1;
}

function getData() {
    return immutableDataset.value
}

function toggleTable() {
    mutableConfig.value.showTable = !mutableConfig.value.showTable;
}

const isAnnotator = ref(false);
function toggleAnnotator() {
    isAnnotator.value = !isAnnotator.value;
}

async function getImage({ scale = 2} = {}) {
    if (!dumbbellChart.value) return;
    const { width, height } = dumbbellChart.value.getBoundingClientRect();
    const aspectRatio = width / height;
    const { imageUri, base64 } = await img({ domElement: dumbbellChart.value, base64: true, img: true, scale })
    return { 
        imageUri, 
        base64, 
        title: FINAL_CONFIG.value.style.chart.title.text,
        width,
        height,
        aspectRatio
    }
}

defineExpose({
    getData,
    getImage,
    generatePdf,
    generateCsv,
    generateImage,
    toggleTable,
    toggleAnnotator,
    toggleFullscreen
});

</script>

<template>
    <div ref="dumbbellChart" :class="`vue-ui-dumbbell ${isFullscreen ? 'vue-data-ui-wrapper-fullscreen' : ''}`" :style="`font-family:${FINAL_CONFIG.style.fontFamily};width:100%; text-align:center;background:${FINAL_CONFIG.style.chart.backgroundColor};${FINAL_CONFIG.responsive ? 'height:100%': ''}`" :id="`dumbbell_${uid}`" @mouseenter="() => setUserOptionsVisibility(true)" @mouseleave="() => setUserOptionsVisibility(false)">
        <PenAndPaper
            v-if="FINAL_CONFIG.userOptions.buttons.annotator"
            :svgRef="svgRef"
            :backgroundColor="FINAL_CONFIG.style.chart.backgroundColor"
            :color="FINAL_CONFIG.style.chart.color"
            :active="isAnnotator"
            @close="toggleAnnotator"
        />

        <div
            ref="noTitle"
            v-if="hasOptionsNoTitle" 
            class="vue-data-ui-no-title-space" 
            :style="`height:36px; width: 100%;background:transparent`"
        />

        <div ref="chartTitle" v-if="FINAL_CONFIG.style.chart.title.text" :style="`width:100%;background:transparent;padding-bottom:24px`">
            <Title
                :key="`title_${titleStep}`"
                :config="{
                    title: {
                        cy: 'donut-div-title',
                        ...FINAL_CONFIG.style.chart.title
                    },
                    subtitle: {
                        cy: 'donut-div-subtitle',
                        ...FINAL_CONFIG.style.chart.title.subtitle
                    }
                }"
            />
        </div>

        <UserOptions
            ref="details"
            :key="`user_option_${step}`"
            v-if="FINAL_CONFIG.userOptions.show && isDataset && (keepUserOptionState ? true : userOptionsVisible)"
            :backgroundColor="FINAL_CONFIG.style.chart.backgroundColor"
            :color="FINAL_CONFIG.style.chart.color"
            :isPrinting="isPrinting"
            :isImaging="isImaging"
            :uid="uid"
            :hasPdf="FINAL_CONFIG.userOptions.buttons.pdf"
            :hasXls="FINAL_CONFIG.userOptions.buttons.csv"
            :hasImg="FINAL_CONFIG.userOptions.buttons.img"
            :hasTable="FINAL_CONFIG.userOptions.buttons.table"
            :hasFullscreen="FINAL_CONFIG.userOptions.buttons.fullscreen"
            :isFullscreen="isFullscreen"
            :titles="{ ...FINAL_CONFIG.userOptions.buttonTitles }"
            :chartElement="dumbbellChart"
            :position="FINAL_CONFIG.userOptions.position"
            :hasAnnotator="FINAL_CONFIG.userOptions.buttons.annotator"
            :isAnnotation="isAnnotator"
            :callbacks="FINAL_CONFIG.userOptions.callbacks"
            :printScale="FINAL_CONFIG.userOptions.print.scale"
            @toggleFullscreen="toggleFullscreen"
            @generatePdf="generatePdf"
            @generateCsv="generateCsv"
            @generateImage="generateImage"
            @toggleTable="toggleTable"
            @toggleAnnotator="toggleAnnotator"
            :style="{
                visibility: keepUserOptionState ? userOptionsVisible ? 'visible' : 'hidden' : 'visible'
            }"
        >
            <template #menuIcon="{ isOpen, color }" v-if="$slots.menuIcon">
                <slot name="menuIcon" v-bind="{ isOpen, color }"/>
            </template>
            <template #optionPdf v-if="$slots.optionPdf">
                <slot name="optionPdf" />
            </template>
            <template #optionCsv v-if="$slots.optionCsv">
                <slot name="optionCsv" />
            </template>
            <template #optionImg v-if="$slots.optionImg">
                <slot name="optionImg" />
            </template>
            <template #optionTable v-if="$slots.optionTable">
                <slot name="optionTable" />
            </template>
            <template v-if="$slots.optionFullscreen" template #optionFullscreen="{ toggleFullscreen, isFullscreen }">
                <slot name="optionFullscreen" v-bind="{ toggleFullscreen, isFullscreen }"/>
            </template>
            <template v-if="$slots.optionAnnotator" #optionAnnotator="{ toggleAnnotator, isAnnotator }">
                <slot name="optionAnnotator" v-bind="{ toggleAnnotator, isAnnotator }" />
            </template>
        </UserOptions>

        <svg
            ref="svgRef"
            :xmlns="XMLNS" 
            v-if="isDataset" 
            :class="{ 'vue-data-ui-fullscreen--on': isFullscreen, 'vue-data-ui-fulscreen--off': !isFullscreen }" 
            :viewBox="`0 0 ${drawingArea.absoluteWidth <= 0 ? 10 : drawingArea.absoluteWidth} ${drawingArea.absoluteHeight <= 0 ? 10 : drawingArea.absoluteHeight}`" :style="`max-width:100%; overflow: visible; background:transparent;color:${FINAL_CONFIG.style.chart.color}`"
        >
            <PackageVersion />

            <!-- BACKGROUND SLOT -->
            <foreignObject 
                v-if="$slots['chart-background']"
                :x="drawingArea.left"
                :y="drawingArea.top"
                :width="Math.max(0.1, drawingArea.width)"
                :height="Math.max(0.1, drawingArea.height)"
                :style="{
                    pointerEvents: 'none'
                }"
            >
                <slot name="chart-background"/>
            </foreignObject>
            
            <!-- VERTICAL GRID -->
            <g v-if="FINAL_CONFIG.style.chart.grid.verticalGrid.show">
                <line
                    data-cy="grid-line-y"
                    v-for="(_, i) in scale.ticks"
                    :x1="drawingArea.left + ((i) * drawingArea.width / (scale.ticks.length - 1))"
                    :x2="drawingArea.left + ((i) * drawingArea.width / (scale.ticks.length - 1))"
                    :y1="drawingArea.top"
                    :y2="drawingArea.bottom"
                    :stroke="FINAL_CONFIG.style.chart.grid.verticalGrid.stroke"
                    :stroke-width="FINAL_CONFIG.style.chart.grid.verticalGrid.strokeWidth"
                    :stroke-dasharray="FINAL_CONFIG.style.chart.grid.verticalGrid.strokeDasharray"
                />
            </g>
            <!-- HORIZONTAL GRID -->
            <g v-if="FINAL_CONFIG.style.chart.grid.horizontalGrid.show">
                <line
                    data-cy="grid-line-x"
                    v-for="(_, i) in immutableDataset"
                    :x1="drawingArea.left"
                    :x2="drawingArea.right"
                    :y1="drawingArea.top + (i * baseRowHeight)"
                    :y2="drawingArea.top + (i * baseRowHeight)"
                    :stroke="FINAL_CONFIG.style.chart.grid.horizontalGrid.stroke"
                    :stroke-width="FINAL_CONFIG.style.chart.grid.horizontalGrid.strokeWidth"
                    :stroke-dasharray="FINAL_CONFIG.style.chart.grid.horizontalGrid.strokeDasharray"
                />
                <line
                    data-cy="grid-base-x"
                    :x1="drawingArea.left"
                    :x2="drawingArea.right"
                    :y1="drawingArea.bottom"
                    :y2="drawingArea.bottom"
                    :stroke="FINAL_CONFIG.style.chart.grid.horizontalGrid.stroke"
                    :stroke-width="FINAL_CONFIG.style.chart.grid.horizontalGrid.strokeWidth"
                    :stroke-dasharray="FINAL_CONFIG.style.chart.grid.horizontalGrid.strokeDasharray"
                />
            </g>
            <!-- Y AXIS LABELS -->
            <g v-if="FINAL_CONFIG.style.chart.labels.yAxisLabels.show">
                <text
                    data-cy="label-y-name"
                    v-for="(datapoint, i) in immutableDataset"
                    :x="drawingArea.left - 6 + FINAL_CONFIG.style.chart.labels.yAxisLabels.offsetX"
                    :y="drawingArea.top + (i * baseRowHeight) + (FINAL_CONFIG.style.chart.labels.yAxisLabels.showProgression ? baseRowHeight / 3 : baseRowHeight / 2) + (FINAL_CONFIG.style.chart.labels.yAxisLabels.fontSize / 3)"
                    :font-size="FINAL_CONFIG.style.chart.labels.yAxisLabels.fontSize"
                    :fill="FINAL_CONFIG.style.chart.labels.yAxisLabels.color"
                    :font-weight="FINAL_CONFIG.style.chart.labels.yAxisLabels.bold ? 'bold': 'normal'"
                    text-anchor="end"
                >
                    {{ datapoint.name }}
                </text>
                <template v-if="FINAL_CONFIG.style.chart.labels.yAxisLabels.showProgression">
                    <text
                        data-cy="label-y-value"
                        v-for="(datapoint, i) in immutableDataset"
                        :x="drawingArea.left - 6 + FINAL_CONFIG.style.chart.labels.yAxisLabels.offsetX"
                        :y="drawingArea.top + (i * baseRowHeight) + (baseRowHeight / 1.3) + (FINAL_CONFIG.style.chart.labels.yAxisLabels.fontSize / 3)"
                        :font-size="FINAL_CONFIG.style.chart.labels.yAxisLabels.fontSize"
                        :fill="FINAL_CONFIG.style.chart.labels.yAxisLabels.color"
                        text-anchor="end"
                    >
                        {{ dataLabel({
                            v: 100 * ((datapoint.end / datapoint.start) - 1),
                            s: '%',
                            r: FINAL_CONFIG.style.chart.labels.yAxisLabels.rounding
                        }) }}
                    </text>
                </template>
            </g>
            <!-- X AXIS LABELS -->
            <g v-if="FINAL_CONFIG.style.chart.labels.xAxisLabels.show">
                <text
                    data-cy="label-x"
                    v-for="(tick, i) in scale.ticks"
                    :x="drawingArea.left + (i * (drawingArea.width / (scale.ticks.length - 1)))"
                    :y="drawingArea.bottom + FINAL_CONFIG.style.chart.labels.xAxisLabels.fontSize + FINAL_CONFIG.style.chart.labels.xAxisLabels.offsetY"
                    :font-size="FINAL_CONFIG.style.chart.labels.xAxisLabels.fontSize"
                    :fill="FINAL_CONFIG.style.chart.labels.xAxisLabels.color"
                    :font-weight="FINAL_CONFIG.style.chart.labels.xAxisLabels.bold ? 'bold': 'normal'"
                    text-anchor="middle"
                >
                    {{ applyDataLabel(
                        FINAL_CONFIG.style.chart.labels.formatter,
                        tick,
                        dataLabel({
                            p: FINAL_CONFIG.style.chart.labels.prefix,
                            v: tick,
                            s: FINAL_CONFIG.style.chart.labels.suffix,
                            r: FINAL_CONFIG.style.chart.labels.xAxisLabels.rounding
                        }),
                        { datapoint: tick, seriesIndex: i }
                        ) 
                    }}
                </text>
            </g>

            <!-- PLOTS -->
            <defs>
                <radialGradient :id="`start_grad_${uid}`" fy="30%">
                    <stop offset="10%" :stop-color="lightenHexColor(FINAL_CONFIG.style.chart.plots.startColor, FINAL_CONFIG.style.chart.plots.gradient.intensity / 100)"/>
                    <stop offset="90%" :stop-color="darkenHexColor(FINAL_CONFIG.style.chart.plots.startColor, 0.1)"/>
                    <stop offset="100%" :stop-color="FINAL_CONFIG.style.chart.plots.startColor"/>
                </radialGradient>
                <radialGradient :id="`end_grad_${uid}`" fy="30%">
                    <stop offset="10%" :stop-color="lightenHexColor(FINAL_CONFIG.style.chart.plots.endColor, FINAL_CONFIG.style.chart.plots.gradient.intensity / 100)"/>
                    <stop offset="90%" :stop-color="darkenHexColor(FINAL_CONFIG.style.chart.plots.endColor, 0.1)"/>
                    <stop offset="100%" :stop-color="FINAL_CONFIG.style.chart.plots.endColor"/>
                </radialGradient>
            </defs>
            <g v-for="(plot, i) in mutableDataset">
                <!-- LINK -->
                <defs>
                    <linearGradient :id="`grad_positive_${uid}`" x1="0%" x2="100%" y1="0%" y2="0%">
                        <stop offset="0%" :stop-color="FINAL_CONFIG.style.chart.plots.startColor"/>
                        <stop offset="100%" :stop-color="FINAL_CONFIG.style.chart.plots.endColor"/>
                    </linearGradient>
                    <linearGradient :id="`grad_negative_${uid}`" x1="0%" x2="100%" y1="0%" y2="0%">
                        <stop offset="0%" :stop-color="FINAL_CONFIG.style.chart.plots.endColor"/>
                        <stop offset="100%" :stop-color="FINAL_CONFIG.style.chart.plots.startColor"/>
                    </linearGradient>
                </defs>
                <g v-if="FINAL_CONFIG.style.chart.plots.link.type === 'curved'">
                    <path
                        data-cy="link-curved"
                        :d="`M 
                            ${plot.startX},${plot.y + FINAL_CONFIG.style.chart.plots.radius / 2} 
                            C ${plot.centerX},${plot.y} ${plot.centerX},${plot.y} 
                            ${plot.endX},${plot.y + FINAL_CONFIG.style.chart.plots.radius / 2}
                            L ${plot.endX},${plot.y - FINAL_CONFIG.style.chart.plots.radius / 2}
                            C ${plot.centerX},${plot.y} ${plot.centerX},${plot.y}
                            ${plot.startX},${plot.y - FINAL_CONFIG.style.chart.plots.radius / 2}
                            Z
                        `"
                        :fill="plot.endX > plot.startX ? `url(#grad_positive_${uid})`: `url(#grad_negative_${uid})`"
                    />
                </g>
                <g v-else>
                    <rect
                        data-cy="link-straight"
                        :x="plot.endX > plot.startX ? plot.startX : plot.endX"
                        :y="plot.y - (FINAL_CONFIG.style.chart.plots.link.strokeWidth / 2)"
                        :height="FINAL_CONFIG.style.chart.plots.link.strokeWidth"
                        :width="Math.abs(plot.endX - plot.startX)"
                        :fill="plot.endX > plot.startX ? `url(#grad_positive_${uid})`: `url(#grad_negative_${uid})`"
                    />
                </g>

                <!-- START -->
                <circle
                    data-cy="datapoint-start"
                    :cx="plot.startX"
                    :cy="plot.y"
                    :r="FINAL_CONFIG.style.chart.plots.radius"
                    :fill="FINAL_CONFIG.style.chart.plots.gradient.show ? `url(#start_grad_${uid})` : FINAL_CONFIG.style.chart.plots.startColor"
                    :stroke="FINAL_CONFIG.style.chart.plots.stroke"
                    :stroke-width="FINAL_CONFIG.style.chart.plots.strokeWidth"
                    
                />
                <!-- END -->
                <circle
                    data-cy="datapoint-end"
                    :cx="plot.endX"
                    :cy="plot.y"
                    :r="FINAL_CONFIG.style.chart.plots.radius"
                    :fill="FINAL_CONFIG.style.chart.plots.gradient.show ? `url(#end_grad_${uid})` : FINAL_CONFIG.style.chart.plots.endColor"
                    :stroke="FINAL_CONFIG.style.chart.plots.stroke"
                    :stroke-width="FINAL_CONFIG.style.chart.plots.strokeWidth"
                    
                />
            </g>
            <!-- START LABELS -->
            <g v-if="FINAL_CONFIG.style.chart.labels.startLabels.show">
                <text
                    data-cy="datapoint-label-start"
                    v-for="(plot, i) in mutableDataset"
                    :x="plot.startX"
                    :y="drawingArea.top + ((i + 1) * baseRowHeight) - (FINAL_CONFIG.style.chart.labels.startLabels.fontSize / 3) + FINAL_CONFIG.style.chart.labels.startLabels.offsetY"
                    :fill="FINAL_CONFIG.style.chart.labels.startLabels.useStartColor ? FINAL_CONFIG.style.chart.plots.startColor : FINAL_CONFIG.style.chart.labels.startLabels.color"
                    :font-size="FINAL_CONFIG.style.chart.labels.startLabels.fontSize"
                    text-anchor="middle"
                    
                >
                    {{ applyDataLabel(
                        FINAL_CONFIG.style.chart.labels.formatter,
                        plot.start,
                        dataLabel({
                            p: FINAL_CONFIG.style.chart.labels.prefix,
                            v: plot.start,
                            s: FINAL_CONFIG.style.chart.labels.suffix,
                            r: FINAL_CONFIG.style.chart.labels.startLabels.rounding
                        }),
                        { datapoint: plot, seriesIndex: i }
                        )
                    }}
                </text>
            </g>
            <!-- END LABELS -->
            <g v-if="FINAL_CONFIG.style.chart.labels.endLabels.show">
                <text
                    data-cy="datapoint-label-end"
                    v-for="(plot, i) in mutableDataset"
                    :x="plot.endX"
                    :y="drawingArea.top + (i * baseRowHeight) + FINAL_CONFIG.style.chart.labels.endLabels.fontSize + FINAL_CONFIG.style.chart.labels.endLabels.offsetY"
                    :fill="FINAL_CONFIG.style.chart.labels.endLabels.useEndColor ? FINAL_CONFIG.style.chart.plots.endColor : FINAL_CONFIG.style.chart.labels.endLabels.color"
                    :font-size="FINAL_CONFIG.style.chart.labels.endLabels.fontSize"
                    text-anchor="middle"
                    
                >
                    {{ applyDataLabel(
                        FINAL_CONFIG.style.chart.labels.formatter,
                        plot.end,
                        dataLabel({
                            p: FINAL_CONFIG.style.chart.labels.prefix,
                            v: plot.end,
                            s: FINAL_CONFIG.style.chart.labels.suffix,
                            r: FINAL_CONFIG.style.chart.labels.endLabels.rounding
                        }),
                        { datapoint: plot, seriesIndex: i }
                        )
                    }}
                </text>
            </g>
            <slot name="svg" :svg="drawingArea"/>
        </svg>

        <div v-if="$slots.watermark" class="vue-data-ui-watermark">
            <slot name="watermark" v-bind="{ isPrinting: isPrinting || isImaging }"/>
        </div>

        <Skeleton 
            v-if="!isDataset"
            :config="{
                type: 'dumbbell',
                style: {
                    backgroundColor: FINAL_CONFIG.style.chart.backgroundColor,
                    dumbbell: {
                        color: '#CCCCCC'
                    }
                }
            }"
        />
        
        <div ref="chartLegend">
            <Legend
                v-if="FINAL_CONFIG.style.chart.legend.show && isDataset"
                :key="`legend_${legendStep}`"
                :legendSet="legendSet"
                :config="legendConfig"
            >
                <template #item="{ legend }">
                    <div :style="`display:flex;align-items:center;gap:4px;font-size:${FINAL_CONFIG.style.chart.legend.fontSize}px`">
                        <svg :xmlns="XMLNS" viewBox="0 0 20 20" :height="FINAL_CONFIG.style.chart.legend.fontSize" :width="FINAL_CONFIG.style.chart.legend.fontSize">
                            <circle :cx="10" :cy="10" :r="9" :fill="legend.color"/>
                        </svg>
                        {{ legend.name }}
                    </div>
                </template>
            </Legend>
            <slot v-else name="legend" v-bind:legend="legendSet" />
        </div>

        <div v-if="$slots.source" ref="source" dir="auto">
            <slot name="source" />
        </div>

        <Accordion hideDetails v-if="isDataset" :config="{
            open: mutableConfig.showTable,
            maxHeight: 10000,
            body: {
                backgroundColor: FINAL_CONFIG.style.chart.backgroundColor,
                color: FINAL_CONFIG.style.chart.color,
            },
            head: {
                backgroundColor: FINAL_CONFIG.style.chart.backgroundColor,
                color: FINAL_CONFIG.style.chart.color,
            }
        }">
            <template #content>
                <DataTable
                    :key="`table_${tableStep}`"
                    :colNames="dataTable.colNames"
                    :head="dataTable.head" 
                    :body="dataTable.body"
                    :config="dataTable.config"
                    :title="`${FINAL_CONFIG.style.chart.title.text}${FINAL_CONFIG.style.chart.title.subtitle.text ? ` : ${FINAL_CONFIG.style.chart.title.subtitle.text}` : ''}`"
                    @close="mutableConfig.showTable = false"
                >
                    <template #th="{ th }">
                        <div v-html="th" style="display:flex;align-items:center"></div>
                    </template>
                    <template #td="{ td }">
                        {{ td.name || td }}
                    </template>
                </DataTable>
            </template>
        </Accordion>
    </div>
</template>

<style scoped>
@import "../vue-data-ui.css";

.vue-ui-dumbbell * {
    transition: unset;
}

.vue-ui-dumbbell {
    user-select: none;
    position: relative;
}
</style>