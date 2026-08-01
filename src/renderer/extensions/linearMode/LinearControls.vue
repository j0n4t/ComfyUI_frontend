<script setup lang="ts">
import { useTimeout } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { ref, useTemplateRef } from 'vue'
import { useI18n } from 'vue-i18n'

import AppModeWidgetList from '@/components/builder/AppModeWidgetList.vue'
import Loader from '@/components/loader/Loader.vue'
import Button from '@/components/ui/button/Button.vue'
import { useTelemetry } from '@/platform/telemetry'
import { useWorkflowStore } from '@/platform/workflow/management/stores/workflowStore'
import PartnerNodesList from '@/renderer/extensions/linearMode/PartnerNodesList.vue'
import { useCommandStore } from '@/stores/commandStore'
import { useQueueSettingsStore } from '@/stores/queueStore'
import { useAppMode } from '@/composables/useAppMode'
import { useAppModeStore } from '@/stores/appModeStore'

const { t } = useI18n()
const commandStore = useCommandStore()
const { batchCount } = storeToRefs(useQueueSettingsStore())
const workflowStore = useWorkflowStore()
const { isBuilderMode } = useAppMode()
const appModeStore = useAppModeStore()
const { hasOutputs } = storeToRefs(appModeStore)


const { toastTo, mobile } = defineProps<{
  toastTo?: string | HTMLElement
  mobile?: boolean
}>()
defineEmits<{ navigateOutputs: [] }>()
defineExpose({ runButtonClick, handleDragDrop })

//NOTE: due to batching, will never be greater than 2
const pendingJobQueues = ref(0)
const { ready: jobToastTimeout, start: resetJobToastTimeout } = useTimeout(
  8000,
  { controls: true, immediate: false }
)
const widgetListRef = useTemplateRef('widgetListRef')

//TODO: refactor out of this file.
//code length is small, but changes should propagate
async function runButtonClick(e: Event) {
  try {
    pendingJobQueues.value += 1
    resetJobToastTimeout()
    const isShiftPressed = 'shiftKey' in e && e.shiftKey
    const commandId = isShiftPressed
      ? 'Comfy.QueuePromptFront'
      : 'Comfy.QueuePrompt'

    if (batchCount.value > 1) {
      useTelemetry()?.trackUiButtonClicked({
        button_id: 'queue_run_multiple_batches_submitted',
        element_group: 'app_mode'
      })
    }
    await commandStore.execute(commandId, {
      metadata: {
        subscribe_to_run: false,
        trigger_source: 'linear'
      }
    })
  } finally {
    //TODO: Error state indicator for failed queue?
    pendingJobQueues.value -= 1
  }
}
function handleDragDrop() {
  return widgetListRef.value?.handleDragDrop()
}
</script>
<template>
  <div
    v-if="!isBuilderMode && hasOutputs"
    class="flex h-full min-w-80 flex-col"
    v-bind="$attrs"
  >
    <section
      v-if="!mobile"
      data-testid="linear-workflow-info"
      class="flex h-12 items-center gap-2 border-x border-border-subtle bg-comfy-menu-bg px-4 py-2 contain-size"
    >
      <span
        class="truncate font-bold"
        v-text="workflowStore.activeWorkflow?.filename"
      />
      <div class="flex-1" />
      <Button
        variant="primary"
        class="h-full text-sm"
        size="lg"
        @click="runButtonClick"
      >
        <i class="icon-[lucide--play]" />
        {{ t('menu.run') }}
      </Button>
    </section>
    <div
      class="flex h-full flex-col gap-2 border-x border-(--interface-stroke) bg-comfy-menu-bg px-2 md:border-y"
    >
      <section
        data-testid="linear-widgets"
        class="grow scroll-shadows-comfy-menu-bg overflow-y-auto contain-size"
      >
        <AppModeWidgetList ref="widgetListRef" :mobile />
      </section>
      <Teleport
        v-if="!jobToastTimeout || pendingJobQueues > 0"
        defer
        :disabled="mobile"
        :to="toastTo"
      >
        <div
          class="flex h-10 items-center gap-2 rounded-lg bg-base-foreground p-1 pr-2 text-base-background md:h-8 md:bg-secondary-background md:text-base-foreground"
        >
          <template v-if="pendingJobQueues === 0">
            <i
              class="icon-[lucide--check] size-5 not-md:bg-success-background"
            />
            <span class="mr-auto" v-text="t('queue.jobAddedToQueue')" />
            <Button
              v-if="mobile"
              variant="inverted"
              @click="$emit('navigateOutputs')"
            >
              {{ t('linearMode.viewJob') }}
            </Button>
          </template>
          <template v-else>
            <Loader size="sm" />
            <span v-text="t('queue.jobQueueing')" />
          </template>
        </div>
      </Teleport>
      <PartnerNodesList v-if="!mobile" />
    </div>
  </div>
  <div
    v-else-if="mobile"
    class="flex size-full items-center bg-base-background p-4 text-center"
    v-text="t('linearMode.mobileNoWorkflow')"
  />
</template>
