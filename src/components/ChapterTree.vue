<template>
  <div class="chapter-tree-container">
    <!-- 顶部提示栏 -->
    <el-alert type="info" :closable="false" show-icon>
      <template #title>分类、章、节上下拖拽调整顺序。</template>
    </el-alert>

    <!-- 顶部操作栏 -->
    <div class="top-action-bar">
      <el-button type="primary" size="small" @click="handleAddChapter">添加分类</el-button>
      <el-button type="primary" size="small" @click="handleBatchEdit">批量修改</el-button>
    </div>

    <!-- 树组件 -->
    <div class="tree-content">
      <el-tree ref="treeRef" :key="treeKey" :data="treeData" :props="treeProps" draggable :default-expand-all="true"
        :highlight-current="true" node-key="id" class="chapter-tree" @node-click="handleNodeClick"
        @node-drop="handleNodeDrop">
        <template #default="{ node, data }">
          <div :class="['tree-node-row', `level-${node.level}`, {
            'is-editing': editingId === data.id,
            'is-selected': selectedNode?.id === data.id,
            'is-disabled': isNodeDisabled(data)
          }]">
            <div class="node-name">
              <template v-if="editingId === data.id">
                <el-input ref="editInputRef" v-model="editText" size="small" @blur="finishEdit"
                  @keyup.enter="finishEdit" @keyup.esc="cancelEdit" class="edit-input" />
              </template>
              <template v-else>
                <span class="editable-text" @dblclick.prevent.stop="handleNodeDoubleClick(data)" title="双击编辑">{{
                  getNodeLabel(node, data) }}</span>
              </template>
            </div>
            <div class="node-actions">
              <slot name="node-actions" :node="node" :data="data" :actions="{
                close: handleClose,
                edit: handleEditNode,
                add: handleAddSubNode,
                manage: handleManage,
                delete: handleDeleteNode
              }">
                <template v-if="actions !== false">
                  <el-button v-if="actions.close" size="small" link :type="isNodeDisabled(data) ? 'success' : 'info'"
                    @click.stop="handleClose(data)">
                    {{ isNodeDisabled(data) ? '启用' : '关闭' }}
                  </el-button>
                  <el-button v-if="actions.edit" size="small" link type="primary"
                    @click.stop="handleEditNode(data)">编辑</el-button>
                  <el-button v-if="actions.add" size="small" link type="primary"
                    @click.stop="handleAddSubNode(data)">添加节</el-button>
                  <el-button v-if="actions.delete" size="small" link type="danger"
                    @click.stop="handleDeleteNode(data)">删除</el-button>
                </template>
              </slot>
            </div>
          </div>
        </template>
      </el-tree>
    </div>

    <!-- 添加章节对话框 -->
    <el-dialog v-model="showAddDialog" title="添加章节" width="450px" :close-on-click-modal="false" @close="closeAddDialog">
      <div class="dialog-content">
        <el-input v-model="newChapterTitle" placeholder="请输入章节标题" clearable @keyup.enter="addAndContinue"
          ref="dialogInputRef" style="flex:1" />
        <el-button type="primary" @click="addAndContinue" :disabled="!newChapterTitle.trim()">添加</el-button>
      </div>

      <!-- 已添加的章节列表 -->
      <div v-if="addedChapters.length > 0" class="added-list">
        <div class="text-xs text-gray-400 mb-2">已添加 ({{ addedChapters.length }})</div>
        <div class="added-list-items">
          <div v-for="(chapter, index) in addedChapters" :key="chapter.id" class="added-item">
            <span class="text-xs text-gray-400 w-5">{{ index + 1 }}.</span>
            <span class="added-item-name">{{ chapter.label }}</span>
            <el-button size="small" link type="danger" @click="removeAddedChapter(index)">删除</el-button>
          </div>
        </div>
      </div>

      <template #footer>
        <el-button @click="showAddDialog = false">取消</el-button>
        <el-button type="primary" @click="confirmAllChapters" :disabled="addedChapters.length === 0">
          确定添加 ({{ addedChapters.length }})
        </el-button>
      </template>
    </el-dialog>

    <!-- 批量修改对话框 -->
    <el-dialog v-model="showBatchEditDialog" title="批量修改章节" width="500px" :close-on-click-modal="false"
      @close="closeBatchEditDialog">
      <div class="batch-edit-content">
        <p class="text-sm text-gray-600 mb-4">选择要修改的章节，输入新名称后点击确认：</p>

        <div class="batch-edit-list">
          <div v-for="(item, index) in batchEditItems" :key="index" class="batch-edit-item">
            <label class="batch-edit-original">{{ item.originalLabel }}</label>
            <span class="text-gray-300">→</span>
            <el-input v-model="item.newLabel" placeholder="输入新名称" size="small" clearable />
            <el-checkbox v-model="item.checked" />
          </div>
        </div>

        <div class="flex gap-2 mt-3">
          <el-button size="small" @click="selectAllForEdit">全选</el-button>
          <el-button size="small" @click="deselectAllForEdit">取消全选</el-button>
        </div>
      </div>

      <template #footer>
        <el-button @click="showBatchEditDialog = false">取消</el-button>
        <el-button type="primary" @click="confirmBatchEdit" :disabled="getCheckedCount() === 0">
          确定修改 ({{ getCheckedCount() }})
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, watch } from 'vue'
import { ElTree, ElMessage, ElMessageBox } from 'element-plus'

export interface TreeNode {
  id: string | number
  label: string
  children?: TreeNode[]
  [key: string]: any
}

export interface TreeAction {
  action: 'add' | 'delete' | 'update' | 'drag' | 'copy' | 'toggle' | 'batchEdit'
  data: TreeNode
  extra?: {
    parentId?: string | number
    targetId?: string | number
    position?: string
    items?: { id: string | number; oldLabel: string; newLabel: string }[]
    disabled?: boolean
  }
}

interface BatchEditItem {
  id: string | number
  originalLabel: string
  newLabel: string
  checked: boolean
}

interface NodeActions {
  close?: boolean
  share?: boolean
  edit?: boolean
  add?: boolean
  manage?: boolean
  delete?: boolean
}

const props = withDefaults(defineProps<{
  treeData: TreeNode[]
  showIndex?: boolean
  levelNames?: string[]
  actions?: NodeActions | false
}>(), {
  showIndex: true,
  levelNames: () => ['章', '节', '小节', '条', '款'],
  actions: () => ({
    close: true,
    edit: true,
    add: true,
    delete: true,
    share: false,
    manage: false
  })
})

const emit = defineEmits<{
  (e: 'update:treeData', data: TreeNode[]): void
  (e: 'action', action: TreeAction): void
  (e: 'add', node: TreeNode, parentId?: string | number): void
  (e: 'delete', node: TreeNode): void
  (e: 'update', node: TreeNode): void
  (e: 'copy', node: TreeNode): void
  (e: 'drag', data: { from: TreeNode; to: TreeNode; position: string }): void
  (e: 'select', node: TreeNode): void
}>()

const treeRef = ref<InstanceType<typeof ElTree> | null>(null)
const editInputRef = ref<any>(null)
const dialogInputRef = ref<any>(null)

const selectedNode = ref<TreeNode | null>(null)
const editingId = ref<string | number | null>(null)
const editText = ref('')
const treeKey = ref(0)

// 对话框相关
const showAddDialog = ref(false)
const newChapterTitle = ref('')
const addedChapters = ref<TreeNode[]>([])

// 批量修改相关
const showBatchEditDialog = ref(false)
const batchEditItems = ref<BatchEditItem[]>([])

// 禁用节点列表
const disabledNodeIds = ref<(string | number)[]>([])

const isNodeDisabled = (data: TreeNode): boolean => disabledNodeIds.value.includes(data.id)

const treeProps = {
  children: 'children',
  label: 'label'
}

// 刷新树组件，强制重新渲染以更新序号
const refreshTree = (): void => {
  treeKey.value++
}

// 获取节点标签（带动态序号）
const getNodeLabel = (node: any, data: TreeNode): string => {
  if (props.showIndex === false) {
    return data.label
  }

  const level = node.level
  let index = 1

  if (node.parent && node.parent.childNodes) {
    index = node.parent.childNodes.findIndex((child: any) => child.data.id === data.id) + 1
  } else if (node.root && node.root.childNodes) {
    index = node.root.childNodes.findIndex((child: any) => child.data.id === data.id) + 1
  }

  if (level === 1) {
    return `第${index}${props.levelNames[0] || '章'}：${data.label}`
  } else if (level === 2) {
    return `第${index}${props.levelNames[1] || '节'}：${data.label}`
  } else {
    const levelName = props.levelNames[level - 1] || '项'
    return `第${index}${levelName}：${data.label}`
  }
}

// 更新所有节点序号（递归）
const updateNodeIndexes = (nodes: TreeNode[], level: number = 1): void => {
  nodes.forEach((node, index) => {
    node._level = level
    node._index = index + 1

    if (node.children && node.children.length > 0) {
      updateNodeIndexes(node.children, level + 1)
    }
  })
}

// 生成唯一ID
const generateId = (): string => {
  return `node_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
}

// 打开添加章节对话框
const handleAddChapter = (): void => {
  newChapterTitle.value = ''
  addedChapters.value = []
  showAddDialog.value = true
  nextTick(() => {
    setTimeout(() => {
      dialogInputRef.value?.focus()
    }, 100)
  })
}

// 添加到临时列表（继续添加）
const addAndContinue = (): void => {
  const title = newChapterTitle.value.trim()
  if (!title) {
    ElMessage.warning('请输入章节标题')
    return
  }

  if (addedChapters.value.some(ch => ch.label === title)) {
    ElMessage.warning('该章节已存在')
    return
  }

  addedChapters.value.push({
    id: generateId(),
    label: title,
    children: []
  })

  newChapterTitle.value = ''

  dialogInputRef.value?.focus()
}

// 从列表中移除
const removeAddedChapter = (index: number): void => {
  addedChapters.value.splice(index, 1)
}

// 确认添加所有章节
const confirmAllChapters = (): void => {
  if (addedChapters.value.length === 0) return

  let newData: TreeNode[]

  if (selectedNode.value) {
    const addChildToNode = (nodes: TreeNode[], targetId: string | number): boolean => {
      for (const node of nodes) {
        if (node.id === targetId) {
          if (!node.children) node.children = []
          node.children.push(...addedChapters.value)
          return true
        }
        if (node.children && addChildToNode(node.children, targetId)) {
          return true
        }
      }
      return false
    }

    newData = JSON.parse(JSON.stringify(props.treeData))
    addChildToNode(newData, selectedNode.value?.id)

    addedChapters.value.forEach(chapter => {
      emit('add', chapter, selectedNode.value?.id)
      emit('action', { action: 'add', data: chapter, extra: { parentId: selectedNode.value?.id } })
    })
  } else {
    newData = [...props.treeData, ...addedChapters.value]

    addedChapters.value.forEach(chapter => {
      emit('add', chapter)
      emit('action', { action: 'add', data: chapter })
    })
  }

  updateNodeIndexes(newData)
  emit('update:treeData', newData)

  showAddDialog.value = false
  refreshTree()
}

// 关闭对话框
const closeAddDialog = (): void => {
  newChapterTitle.value = ''
  addedChapters.value = []
  showAddDialog.value = false
}


// ========== 批量修改功能 ==========

// 收集选中节点及其子节点用于批量修改
const collectSelectedNodes = (nodes: TreeNode[], targetId: string | number, result: BatchEditItem[]): boolean => {
  for (const node of nodes) {
    if (node.id === targetId) {
      // 找到目标节点，收集它和所有子节点
      collectNodeAndChildren(node, result)
      return true
    }
  }
  return false
}

const collectNodeAndChildren = (node: TreeNode, result: BatchEditItem[], level: number = 1, index: number = 1): void => {
  let fullLabel: string

  if (props.showIndex === false) {
    fullLabel = node.label
  } else {
    let prefix = ''
    if (level === 1) {
      prefix = `第${index}${props.levelNames[0] || '章'}：`
    } else if (level === 2) {
      prefix = `第${index}${props.levelNames[1] || '节'}：`
    } else {
      const levelName = props.levelNames[level - 1] || '项'
      prefix = `第${index}${levelName}：`
    }
    fullLabel = prefix + node.label
  }

  result.push({
    id: node.id,
    originalLabel: fullLabel,
    newLabel: node.label,
    checked: true
  })

  if (node.children && node.children.length > 0) {
    node.children.forEach((child, idx) => {
      collectNodeAndChildren(child, result, level + 1, idx + 1)
    })
  }
}

// 打开批量修改对话框（只处理选中节点及其子节点）
const handleBatchEdit = (): void => {
  if (!selectedNode.value) {
    ElMessage.warning('请先选择一个节点')
    return
  }

  batchEditItems.value = []

  if (!collectSelectedNodes(props.treeData, selectedNode.value.id, batchEditItems.value)) {
    ElMessage.warning('未找到选中的节点')
    return
  }

  showBatchEditDialog.value = true
}

// 全选
const selectAllForEdit = (): void => {
  batchEditItems.value.forEach(item => item.checked = true)
}

// 取消全选
const deselectAllForEdit = (): void => {
  batchEditItems.value.forEach(item => item.checked = false)
}

// 获取选中数量
const getCheckedCount = (): number => {
  return batchEditItems.value.filter(item => item.checked && item.newLabel.trim()).length
}

// 确认批量修改
const confirmBatchEdit = (): void => {
  const checkedItems = batchEditItems.value.filter(item => item.checked && item.newLabel.trim())

  if (checkedItems.length === 0) {
    ElMessage.warning('请至少选择一项并输入新名称')
    return
  }

  // 检查是否有重复
  const newLabels = checkedItems.map(item => item.newLabel.trim())
  const uniqueLabels = new Set(newLabels)
  if (newLabels.length !== uniqueLabels.size) {
    ElMessage.warning('存在重复的新名称，请检查')
    return
  }

  const newData = JSON.parse(JSON.stringify(props.treeData))

  // 执行修改
  const batchItems: { id: string | number; oldLabel: string; newLabel: string }[] = []

  const applyChanges = (nodes: TreeNode[]): boolean => {
    for (const node of nodes) {
      const item = checkedItems.find(i => i.id === node.id)
      if (item) {
        const oldLabel = node.label
        node.label = item.newLabel.trim()
        emit('update', node)
        batchItems.push({ id: node.id, oldLabel, newLabel: item.newLabel.trim() })
      }
      if (node.children && applyChanges(node.children)) {
        continue
      }
    }
    return false
  }

  applyChanges(newData)
  updateNodeIndexes(newData)
  emit('update:treeData', newData)

  emit('action', {
    action: 'batchEdit',
    data: selectedNode.value || {} as TreeNode,
    extra: { items: batchItems }
  })

  ElMessage.success(`成功修改 ${checkedItems.length} 个章节`)
  showBatchEditDialog.value = false
  refreshTree()
}

// 关闭批量修改对话框
const closeBatchEditDialog = (): void => {
  batchEditItems.value = []
  showBatchEditDialog.value = false
}

// ========== 节点操作 ==========

// 关闭/启用节点
const handleClose = (data: TreeNode): void => {
  const index = disabledNodeIds.value.indexOf(data.id)
  if (index > -1) {
    disabledNodeIds.value.splice(index, 1)
    ElMessage.success('已启用该节点')
    emit('action', { action: 'toggle', data, extra: { disabled: false } })
  } else {
    disabledNodeIds.value.push(data.id)
    ElMessage.info('已禁用该节点')
    emit('action', { action: 'toggle', data, extra: { disabled: true } })
  }
}

const handleShare = (_data: TreeNode): void => { }

const handleManage = (_data: TreeNode): void => { }

// 节点点击
const handleNodeClick = (data: TreeNode, node: any): void => {
  selectedNode.value = data
  if (node.parent && node.parent.data) {
    selectedNode.value._parentId = node.parent.data.id
  } else {
    delete selectedNode.value._parentId
  }
  selectedNode.value._level = node.level
  emit('select', data)
}

// 双击节点进入编辑模式
const handleNodeDoubleClick = (data: TreeNode): void => {
  editingId.value = data.id
  editText.value = data.label
  nextTick(() => {
    editInputRef.value?.focus()
    editInputRef.value?.select()
  })
}

// 编辑节点（按钮触发）
const handleEditNode = (data: TreeNode): void => {
  editingId.value = data.id
  editText.value = data.label
  nextTick(() => {
    editInputRef.value?.focus()
    editInputRef.value?.select()
  })
}

// 完成编辑
const finishEdit = (): void => {
  if (editingId.value === null) return

  const newText = editText.value.trim()
  if (!newText) {
    ElMessage.warning('节点名称不能为空')
    return
  }

  const updateNode = (nodes: TreeNode[]): boolean => {
    for (const node of nodes) {
      if (node.id === editingId.value) {
        const oldLabel = node.label

        if (newText === oldLabel) {
          editingId.value = null
          return false
        }

        node.label = newText
        emit('update', node)
        emit('action', { action: 'update', data: { ...node, _oldLabel: oldLabel } })
        return true
      }
      if (node.children && updateNode(node.children)) {
        return true
      }
    }
    return false
  }

  const newData = JSON.parse(JSON.stringify(props.treeData))
  if (updateNode(newData)) {
    emit('update:treeData', newData)
  }
  editingId.value = null
}

// 取消编辑
const cancelEdit = (): void => {
  editingId.value = null
}

// 删除节点
const handleDeleteNode = (data: TreeNode): void => {
  ElMessageBox.confirm(
    '确定要删除该节点及其所有子节点吗？',
    '提示',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    }
  ).then(() => {
    const removeNode = (nodes: TreeNode[]): boolean => {
      const index = nodes.findIndex((node) => node.id === data.id)
      if (index !== -1) {
        nodes.splice(index, 1)
        return true
      }
      for (const node of nodes) {
        if (node.children && removeNode(node.children)) {
          return true
        }
      }
      return false
    }

    const newData = JSON.parse(JSON.stringify(props.treeData))
    removeNode(newData)
    updateNodeIndexes(newData)
    emit('update:treeData', newData)
    emit('delete', data)
    emit('action', { action: 'delete', data })
    refreshTree()
  }).catch(() => { })
}

// 添加子节点（按钮触发 - 调用添加章节对话框）
const handleAddSubNode = (data: TreeNode): void => {
  selectedNode.value = { ...data }
  handleAddChapter()
}

// 拖拽放置
const handleNodeDrop = (
  draggingNode: any,
  dropNode: any,
  dropType: string
): void => {
  if (!draggingNode || !dropNode) return

  const sourceData = draggingNode.data
  const targetData = dropNode.data
  const position = dropType

  console.log('[拖拽事件]', {
    from: sourceData.label,
    to: targetData.label,
    position: position
  })

  const newData = JSON.parse(JSON.stringify(props.treeData))

  const removeNode = (nodes: TreeNode[], targetId: string | number): TreeNode | null => {
    const index = nodes.findIndex((node) => node.id === targetId)
    if (index !== -1) {
      return nodes.splice(index, 1)[0]
    }
    for (const node of nodes) {
      if (node.children) {
        const removed = removeNode(node.children, targetId)
        if (removed) return removed
      }
    }
    return null
  }

  const removedNode = removeNode(newData, sourceData.id)
  if (!removedNode) return

  if (position === 'before' || position === 'after') {
    const addToSibling = (nodes: TreeNode[], targetId: string | number): boolean => {
      const index = nodes.findIndex((node) => node.id === targetId)
      if (index !== -1) {
        const insertIndex = position === 'before' ? index : index + 1
        nodes.splice(insertIndex, 0, removedNode)
        return true
      }
      for (const node of nodes) {
        if (node.children && addToSibling(node.children, targetId)) {
          return true
        }
      }
      return false
    }
    addToSibling(newData, targetData.id)
  } else {
    const addAsChild = (nodes: TreeNode[], targetId: string | number): boolean => {
      for (const node of nodes) {
        if (node.id === targetId) {
          if (!node.children) node.children = []
          node.children.push(removedNode)
          return true
        }
        if (node.children && addAsChild(node.children, targetId)) {
          return true
        }
      }
      return false
    }
    addAsChild(newData, targetData.id)
  }

  updateNodeIndexes(newData)
  emit('update:treeData', newData)
  emit('drag', {
    from: sourceData,
    to: targetData,
    position
  })
  emit('action', {
    action: 'drag',
    data: sourceData,
    extra: { targetId: targetData.id, position }
  })

  refreshTree()
}

// 暴露方法
const getTreeData = (): TreeNode[] => {
  return props.treeData
}

const setTreeData = (data: TreeNode[]): void => {
  updateNodeIndexes(data)
  emit('update:treeData', data)
  refreshTree()
}

const expandAll = (): void => {
  (treeRef.value as any)?.expandAll()
}

const collapseAll = (): void => {
  (treeRef.value as any)?.collapseAll()
}

// 监听数据变化，自动刷新
watch(
  () => props.treeData,
  (newData) => {
    if (newData && newData.length > 0) {
      updateNodeIndexes([...newData])
    }
  },
  { immediate: true, deep: true }
)

defineExpose({
  getTreeData,
  setTreeData,
  expandAll,
  collapseAll,
  handleClose: (data: TreeNode) => handleClose(data),
  handleShare: (data: TreeNode) => handleShare(data),
  handleEditNode: (data: TreeNode) => handleEditNode(data),
  handleAddSubNode: (data: TreeNode) => handleAddSubNode(data),
  handleManage: (data: TreeNode) => handleManage(data),
  handleDeleteNode: (data: TreeNode) => handleDeleteNode(data)
})
</script>

<style scoped>
.chapter-tree-container {
  border: 1px solid #dcdcdc;
  background: #fff;
}

.top-action-bar {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  border-bottom: 1px solid #dcdcdc;
  gap: 6px;
}

.tree-content {
  min-height: 300px;
}

.chapter-tree {
  width: 100%;
}

.chapter-tree :deep(.el-tree-node) {
  padding: 0;
}

.chapter-tree :deep(.el-tree-node__content) {
  padding: 0;
  height: 40px;
  line-height: 40px;
  border-bottom: 1px solid #ebeef5;
  transition: background-color 0.15s;
  position: relative;
}

.chapter-tree :deep(.el-tree-node__content:hover) {
  filter: brightness(0.97);
}

.chapter-tree :deep(.el-tree--highlight-current .el-tree-node.is-current > .el-tree-node__content) {
  filter: brightness(0.95);
}

/* 节点行 */
.tree-node-row {
  display: flex;
  width: 100%;
  height: 100%;
  align-items: center;
  border-left: 3px solid transparent;
}

.tree-node-row.is-editing {
  background-color: #ecf5ff !important;
}

.tree-node-row.is-selected {
  border-left: 3px solid #409eff;
}

.tree-node-row.is-selected .node-name span {
  color: #1890ff;
  font-weight: bold;
}

.tree-node-row.is-disabled {
  opacity: 0.5;
  background-color: #f5f5f5 !important;
}

.tree-node-row.is-disabled .node-name span {
  color: #999 !important;
  text-decoration: line-through;
}

.tree-node-row.is-disabled .node-actions :deep(.el-button) {
  pointer-events: none;
  opacity: 0.4;
}

/* 层级颜色 */
.tree-node-row.level-1 {
  background-color: #f5f3f3;
}

.tree-node-row.level-2 {
  background-color: #f5f9ff;
}

.tree-node-row.level-3 {
  background-color: #fffef5;
}

.tree-node-row.level-4 {
  background-color: #f5fff5;
}

.tree-node-row.level-5 {
  background-color: #faf5ff;
}

.node-name {
  flex: 1;
  display: flex;
  align-items: center;
  padding-left: 8px;
  min-width: 0;
  overflow: hidden;
  font-size: 13px;
  color: #333;
}

.node-name span {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.editable-text {
  cursor: pointer;
  padding: 2px 4px;
  border-radius: 3px;
  transition: all 0.15s;
}

.editable-text:hover {
  background-color: rgba(64, 158, 255, 0.1);
  color: #409eff;
}

.edit-input {
  width: 100%;
}

.node-actions {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 4px;
  padding-right: 12px;
}

.chapter-tree :deep(.el-tree-node__expand-icon) {
  padding: 0 6px;
  font-size: 14px;
  color: #c0c4cc;
}

.chapter-tree :deep(.el-tree-node__expand-icon:hover) {
  color: #409eff;
}

.chapter-tree :deep(.el-tree-node__label) {
  width: 100%;
}

/* 拖拽效果 */
.chapter-tree :deep(.el-tree-node.is-dragging > .el-tree-node__content) {
  border: 2px dashed #409eff !important;
  background-color: rgba(64, 158, 255, 0.08) !important;
  opacity: 0.7;
}

.chapter-tree :deep(.el-tree-node.is-drag-over:not(.is-dragging) > .el-tree-node__content) {
  background-color: rgba(64, 158, 255, 0.12) !important;
}

.chapter-tree :deep(.el-tree-node.is-drop-inner:not(.is-dragging) > .el-tree-node__content) {
  border: 2px dashed #409eff !important;
  background-color: rgba(64, 158, 255, 0.06) !important;
}

.chapter-tree :deep(.el-tree-node.is-drop-before:not(.is-dragging) > .el-tree-node__content) {
  box-shadow: inset 3px 0 0 0 #409eff;
}

.chapter-tree :deep(.el-tree-node.is-drop-after:not(.is-dragging) > .el-tree-node__content) {
  box-shadow: inset 0 -3px 0 0 #409eff;
}

.chapter-tree :deep(.el-tree-node.is-drag-over .el-tree-node__expand-icon) {
  color: #409eff;
}

.chapter-tree :deep(.el-tree-node.is-drag-over.is-not-allowed > .el-tree-node__content) {
  border: 2px dashed #f56c6c !important;
  background-color: rgba(245, 108, 108, 0.08) !important;
}

/* 对话框 */
.dialog-content {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 20px;
}

.added-list {
  margin-top: 16px;
  border-top: 1px solid #ebeef5;
  padding-top: 12px;
}

.added-list-items {
  max-height: 180px;
  overflow-y: auto;
}

.added-item {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-radius: 4px;
  margin-bottom: 6px;
  gap: 8px;
}

.added-item:last-child {
  margin-bottom: 0;
}

.added-item-name {
  flex: 1;
  font-size: 13px;
  color: #303133;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.batch-edit-content {
  padding: 10px 0;
}

.batch-edit-list {
  max-height: 350px;
  overflow-y: auto;
  border: 1px solid #ebeef5;
  border-radius: 4px;
  padding: 10px;
}

.batch-edit-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 4px;
  border-bottom: 1px solid #f0f0f0;
}

.batch-edit-item:last-child {
  border-bottom: none;
}

.batch-edit-original {
  font-size: 12px;
  color: #909399;
  min-width: 140px;
  max-width: 180px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.batch-edit-item .el-input {
  flex: 1;
}
</style>
