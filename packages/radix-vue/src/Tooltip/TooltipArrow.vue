<script lang="ts">
import type { PrimitiveProps } from '@/Primitive'

export interface TooltipArrowProps extends PrimitiveProps {
  /**
   * The width of the arrow in pixels.
   *
   * @default 10
   */
  width?: number

  /**
   * The height of the arrow in pixels.
   *
   * @default 5
   */
  height?: number
}
</script>

<script setup lang="ts">
import { PopperArrow } from '@/Popper'

const props = withDefaults(defineProps<TooltipArrowProps>(), {
  width: 10,
  height: 5,
})
</script>

<template>
  <PopperArrow v-bind="props" />
</template>
