<template>
  <TreeExplorer
    v-model:expanded-keys="dummyExpandedKeys"
    :root="renderTreeNode(openWorkflowsTree, WorkflowTreeType.Open)"
    :selection-keys="selectionKeys"
  >
    <template #node="{ node }">
      <TreeExplorerTreeNode :node="node">
        <template #before-label="{ node: treeNode }">
          <span v-if="treeNode.data?.isModified || !treeNode.data?.isPersisted"
            >*</span
          >
        </template>
        <template #actions="{ node: treeNode }">
          <Button
            class="close-workflow-button"
            icon="pi pi-times"
            text
            :severity="workspaceStore.shiftDown ? 'danger' : 'secondary'"
            size="small"
            @click.stop="handleCloseWorkflow(treeNode.data as ComfyWorkflow)"
          />
        </template>
      </TreeExplorerTreeNode>
    </template>
  </TreeExplorer>
</template>
<script setup lang="ts">
import { Button } from 'primevue'
import { computed, ref } from 'vue'

import TreeExplorerTreeNode from '@/components/common/TreeExplorerTreeNode.vue'
import { useTreeExpansion } from '@/composables/useTreeExpansion'
import { t } from '@/i18n'
import { useWorkflowService } from '@/platform/workflow/core/services/workflowService'
import {
  ComfyWorkflow,
  useWorkflowStore
} from '@/platform/workflow/management/stores/workflowStore'
import { useWorkspaceStore } from '@/stores/workspaceStore'
import type { TreeExplorerNode, TreeNode } from '@/types/treeExplorerTypes'
import { appendJsonExt } from '@/utils/formatUtil'
import { buildTree } from '@/utils/treeUtil'

import TreeExplorer from './common/TreeExplorer.vue'

const handleCloseWorkflow = async (workflow?: ComfyWorkflow) => {
  if (workflow) {
    await workflowService.closeWorkflow(workflow, {
      warnIfUnsaved: !workspaceStore.shiftDown
    })
  }
}

const selectionKeys = computed(() => ({
  [`root/${workflowStore.activeWorkflow?.key}`]: true
}))

// Open workflows tree is flat.
const openWorkflowsTree = computed(() =>
  buildTree(workflowStore.openWorkflows, (workflow) => [workflow.key])
)

enum WorkflowTreeType {
  Open = 'Open',
  Bookmarks = 'Bookmarks',
  Browse = 'Browse'
}

const workflowStore = useWorkflowStore()
const workflowService = useWorkflowService()
const workspaceStore = useWorkspaceStore()

const expandedKeys = ref<Record<string, boolean>>({})
const { toggleNodeOnEvent } = useTreeExpansion(expandedKeys)
const dummyExpandedKeys = ref<Record<string, boolean>>({})

const renderTreeNode = (
  node: TreeNode,
  type: WorkflowTreeType
): TreeExplorerNode<ComfyWorkflow> => {
  const children = node.children?.map((child) => renderTreeNode(child, type))

  const workflow: ComfyWorkflow = node.data

  async function handleClick(
    this: TreeExplorerNode<ComfyWorkflow>,
    e: MouseEvent
  ) {
    if (this.leaf) {
      await workflowService.openWorkflow(workflow)
    } else {
      toggleNodeOnEvent(e, this)
    }
  }

  const actions = node.leaf
    ? {
        handleClick,
        async handleRename(newName: string) {
          const newPath =
            type === WorkflowTreeType.Browse
              ? workflow.directory + '/' + appendJsonExt(newName)
              : ComfyWorkflow.basePath + appendJsonExt(newName)

          await workflowService.renameWorkflow(workflow, newPath)
        },
        handleDelete: workflow.isTemporary
          ? undefined
          : async function () {
              await workflowService.deleteWorkflow(workflow)
            },
        contextMenuItems() {
          return [
            {
              label: t('g.insert'),
              icon: 'pi pi-file-export',
              command: async () => {
                const workflow = node.data
                await workflowService.insertWorkflow(workflow)
              }
            },
            {
              label: t('g.duplicate'),
              icon: 'pi pi-file-export',
              command: async () => {
                const workflow = node.data
                await workflowService.duplicateWorkflow(workflow)
              }
            }
          ]
        },
        draggable: true
      }
    : { handleClick }

  return {
    key: node.key,
    label: node.label,
    leaf: node.leaf,
    data: node.data,
    children,
    ...actions
  }
}
</script>
