<script setup>
import BaseIcon from './BaseIcon.vue';
const props = defineProps({
    color: {
        type: String,
        default: '#1A1A1A'
    },
    isFullscreen: {
        type: Boolean,
        default: false
    }
});
const emit = defineEmits(['moveLeft', 'moveTop', 'moveRight', 'moveBottom', 'reset'])
</script>

<template>
    <div :style="`position: ${isFullscreen ? 'fixed' : 'absolute'};bottom:0;right:${isFullscreen ? '12px' : '0'};width:80px;height:80px`" data-dom-to-png-ignore>
        <div style="position: relative;height:100%;width:100%">           
            <button data-cy="direction-pad-left" @click.stop="emit('moveLeft')" style="position: absolute;left:0;top:50%;transform:translateY(-50%);height:24px;width:24px;padding:0;background:transparent;border:none;display:flex;align-items:center;justify-content:center;">
                <BaseIcon :stroke="color" name="arrowLeft" style="cursor: pointer"/>
            </button>
            <button data-cy="direction-pad-top" @click.stop="emit('moveTop')" style="position: absolute;top:0;left:50%;transform:translateX(-50%);height:24px;width:24px;padding:0;background:transparent;border:none;display:flex;align-items:center;justify-content:center;">
                <BaseIcon :stroke="color" name="arrowTop" style="cursor: pointer"/>
            </button>
            <button data-cy="direction-pad-right" @click.stop="emit('moveRight')" style="position: absolute;right:0;top:50%;transform:translateY(-50%);height:24px;width:24px;padding:0;background:transparent;border:none;display:flex;align-items:center;justify-content:center;">
                <BaseIcon :stroke="color" name="arrowRight" style="cursor: pointer"/>
            </button>
            <button data-cy="direction-pad-bottom" @click.stop="emit('moveBottom')" style="position: absolute;bottom:0;left:50%;transform:translateX(-50%);height:24px;width:24px;padding:0;background:transparent;border:none;display:flex;align-items:center;justify-content:center;">
                <BaseIcon :stroke="color" name="arrowBottom" style="cursor: pointer"/>
            </button>
            <button data-cy="direction-pad-reset" @click.stop="emit('reset')" style="position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);height:24px;width:24px;padding:0;background:transparent;border:none;display:flex;align-items:center;justify-content:center;">
                <BaseIcon :stroke="color" name="close" style="cursor: pointer" :strokeWidth="2"/>
            </button>
        </div>
    </div>
</template>