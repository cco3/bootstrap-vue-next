<template>
  <BTransition :no-fade="!fadeBoolean" :trans-props="{enterToClass: 'show'}">
    <div
      v-if="isAlertVisible"
      ref="element"
      class="alert"
      role="alert"
      aria-live="polite"
      aria-atomic="true"
      :class="computedClasses"
    >
      <slot />
      <template v-if="dismissibleBoolean">
        <!-- TODO this renders incorrectly -->
        <BButton v-if="hasCloseSlot || closeContent" v-bind="closeAttrs" @click="closeClicked">
          <slot name="close">
            {{ closeContent }}
          </slot>
        </BButton>
        <BCloseButton v-else :aria-label="dismissLabel" v-bind="closeAttrs" @click="closeClicked" />
      </template>
    </div>
  </BTransition>
</template>

<script setup lang="ts">
import BTransition from '../BTransition/BTransition.vue'
import BCloseButton from '../BButton/BCloseButton.vue'
import BButton from '../BButton/BButton.vue'
import type {Booleanish, ButtonType, ButtonVariant, ColorVariant} from '../../types'
import {computed, onBeforeUnmount, ref, useSlots, watch, watchEffect} from 'vue'
import {useBooleanish, useCountdown} from '../../composables'
import {isEmptySlot} from '../../utils'
import {useElementHover, useToNumber, useVModel} from '@vueuse/core'

const props = withDefaults(
  defineProps<{
    noHoverPause?: Booleanish
    dismissLabel?: string
    dismissible?: Booleanish
    fade?: Booleanish
    closeVariant?: ButtonVariant | null
    modelValue?: boolean | number
    variant?: ColorVariant | null
    closeContent?: string
    immediate?: Booleanish
    interval?: number | string
    showOnPause?: Booleanish
  }>(),
  {
    closeContent: undefined,
    closeVariant: 'secondary',
    noHoverPause: false,
    interval: 1000,
    dismissLabel: 'Close',
    dismissible: false,
    fade: false,
    modelValue: false,
    variant: 'info',
    immediate: true,
    showOnPause: true,
  }
)

const emit = defineEmits<{
  'closed': []
  'close-countdown': [value: number]
  'update:modelValue': [value: boolean | number]
}>()

defineSlots<{
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  default?: (props: Record<string, never>) => any
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  close?: (props: Record<string, never>) => any
}>()

const slots = useSlots()

const element = ref<HTMLElement | null>(null)

const modelValue = useVModel(props, 'modelValue', emit)
const isHovering = useElementHover(element)

const dismissibleBoolean = useBooleanish(() => props.dismissible)
const fadeBoolean = useBooleanish(() => props.fade)
const immediateBoolean = useBooleanish(() => props.immediate)
const showOnPauseBoolean = useBooleanish(() => props.showOnPause)
const noHoverPauseBoolean = useBooleanish(() => props.noHoverPause)
const intervalNumber = useToNumber(() => props.interval)

const hasCloseSlot = computed(() => !isEmptySlot(slots.close))

const countdownLength = computed(() =>
  typeof modelValue.value === 'boolean' ? 0 : modelValue.value
)

const computedClasses = computed(() => ({
  [`alert-${props.variant}`]: props.variant !== null,
  'alert-dismissible': dismissibleBoolean.value,
}))

const {
  isActive,
  pause,
  restart,
  resume,
  stop,
  isPaused,
  value: remainingMs,
} = useCountdown(countdownLength, intervalNumber, {
  immediate: typeof modelValue.value === 'number' && immediateBoolean.value,
})

const isAlertVisible = computed<boolean>(() =>
  typeof modelValue.value === 'boolean'
    ? modelValue.value
    : isActive.value || (showOnPauseBoolean.value && isPaused.value)
)

const closeAttrs = computed(() => ({
  variant: props.closeVariant,
  type: 'button' as ButtonType,
}))

watchEffect(() => {
  emit('close-countdown', remainingMs.value)
})

const closeClicked = (): void => {
  if (typeof modelValue.value === 'boolean') {
    modelValue.value = false
  } else {
    modelValue.value = 0
    stop()
  }
  emit('closed')
}

// TODO mouseleave/mouseenter could be replaced with useElementHover with a watcher
const onMouseEnter = () => {
  if (noHoverPauseBoolean.value) return
  pause()
}

watch(isHovering, (newValue) => {
  if (newValue) {
    onMouseEnter()
    return
  }
  resume()
})

onBeforeUnmount(stop)

defineExpose({pause, resume, restart, stop})
</script>
