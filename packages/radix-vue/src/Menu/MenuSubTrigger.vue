<script lang="ts">
import type { MenuItemImplProps } from './MenuItemImpl.vue'
import type { Side } from './utils'

export interface MenuSubTriggerProps extends MenuItemImplProps {}
</script>

<script setup lang="ts">
import { computed, nextTick, onUnmounted, ref } from 'vue'
import MenuItemImpl from './MenuItemImpl.vue'
import { injectMenuContext, injectMenuRootContext } from './MenuRoot.vue'
import { injectMenuSubContext } from './MenuSub.vue'
import { injectMenuContentContext } from './MenuContentImpl.vue'
import MenuAnchor from './MenuAnchor.vue'
import { SUB_OPEN_KEYS, getOpenState, isMouseEvent } from './utils'

const props = defineProps<MenuSubTriggerProps>()

const menuContext = injectMenuContext()
const rootContext = injectMenuRootContext()
const subContext = injectMenuSubContext()
const contentContext = injectMenuContentContext()

const openTimerRef = ref<number | null>(null)
const pointerGraceTimerRef = computed(
  () => contentContext.pointerGraceTimerRef.value,
)

function clearOpenTimer() {
  if (openTimerRef.value)
    window.clearTimeout(openTimerRef.value)
  openTimerRef.value = null
}

onUnmounted(() => {
  clearOpenTimer()
})

function handlePointerMove(event: PointerEvent) {
  if (!isMouseEvent(event))
    return
  contentContext.onItemEnter(event)
  menuContext.onOpenChange(true)
  if (event.defaultPrevented)
    return
  if (!props.disabled && !menuContext.open.value && !openTimerRef.value) {
    contentContext.onPointerGraceIntentChange(null)
    openTimerRef.value = window.setTimeout(() => {
      clearOpenTimer()
    }, 100)
  }
}

function handlePointerLeave(event: PointerEvent) {
  if (!isMouseEvent(event))
    return
  clearOpenTimer()

  const contentRect = menuContext.content.value?.getBoundingClientRect()

  if (contentRect) {
    // TODO: make sure to update this when we change positioning logic
    const side = menuContext.content.value?.dataset.side as Side

    const rightSide = side === 'right'
    const bleed = rightSide ? -5 : +5
    const contentNearEdge = contentRect[rightSide ? 'left' : 'right']
    const contentFarEdge = contentRect[rightSide ? 'right' : 'left']

    contentContext.onPointerGraceIntentChange({
      area: [
        // Apply a bleed on clientX to ensure that our exit point is
        // consistently within polygon bounds
        { x: event.clientX + bleed, y: event.clientY },
        { x: contentNearEdge, y: contentRect.top },
        { x: contentFarEdge, y: contentRect.top },
        { x: contentFarEdge, y: contentRect.bottom },
        { x: contentNearEdge, y: contentRect.bottom },
      ],
      side,
    })

    window.clearTimeout(pointerGraceTimerRef.value)
    contentContext.pointerGraceTimerRef.value = window.setTimeout(
      () => contentContext.onPointerGraceIntentChange(null),
      300,
    )
  }
  else {
    contentContext.onTriggerLeave(event)
    if (event.defaultPrevented)
      return

    // There's 100ms where the user may leave an item before the submenu was opened.
    contentContext.onPointerGraceIntentChange(null)
  }
}

async function handleKeyDown(event: KeyboardEvent) {
  const isTypingAhead = contentContext.searchRef.value !== ''
  if (props.disabled || (isTypingAhead && event.key === ' '))
    return
  if (SUB_OPEN_KEYS[rootContext.dir.value].includes(event.key)) {
    menuContext.onOpenChange(true)

    await nextTick()
    // The trigger may hold focus if opened via pointer interaction
    // so we ensure content is given focus again when switching to keyboard.
    menuContext.content.value?.focus()
    // prevent window from scrolling
    event.preventDefault()
  }
}

// watchEffect((cleanupFn) => {
//   const pointerGraceTimer = pointerGraceTimerRef.value;
//   cleanupFn(() => {
//     window.clearTimeout(pointerGraceTimer);
//     contentContext?.onPointerGraceIntentChange(null);
//   });
// });
</script>

<template>
  <MenuAnchor as-child>
    <MenuItemImpl
      :id="subContext.triggerId"
      :ref="
        (vnode) => {
          // @ts-ignore
          subContext?.onTriggerChange(vnode?.el);
        }
      "
      aria-haspopup="menu"
      :aria-expanded="menuContext.open.value"
      :aria-controls="subContext.contentId"
      :data-state="getOpenState(menuContext.open.value)"
      @click="
        async (event) => {
          if (props.disabled || event.defaultPrevented) return;
          /**
           * We manually focus because iOS Safari doesn't always focus on click (e.g. buttons)
           * and we rely heavily on `onFocusOutside` for submenus to close when switching
           * between separate submenus.
           */
          event.currentTarget.focus();
          if (!menuContext.open.value) menuContext.onOpenChange(true);
        }
      "
      @pointermove="handlePointerMove"
      @pointerleave="handlePointerLeave"
      @keydown="handleKeyDown"
    >
      <slot />
    </MenuItemImpl>
  </MenuAnchor>
</template>
