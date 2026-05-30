<template>
  <div class="min-h-screen bg-gray-100 p-6">
    <div class="max-w-5xl mx-auto">
      <!-- 操作按钮 -->
      <div class="mb-4 flex items-center gap-4">
        <button @click="fetchTreeData" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
          从远程加载数据
        </button>
        <span v-if="loading" class="text-gray-500">加载中...</span>
        <span v-if="lastLoadTime" class="text-green-600 text-sm">
          上次加载: {{ lastLoadTime }}
        </span>
      </div>

      <!-- 操作日志 -->
      <div v-if="actionLogs.length > 0" class="mb-4 p-3 bg-white rounded shadow text-sm max-h-40 overflow-y-auto">
        <div class="font-bold mb-2 text-gray-700">操作日志：</div>
        <div v-for="(log, index) in actionLogs" :key="index"
          :class="['py-1 px-2 rounded my-1', getLogClass(log.action)]">
          {{ log.time }} - [{{ log.action.toUpperCase() }}] {{ log.message }}
        </div>
      </div>

      <!-- 组件：使用统一的 @action 事件 -->
      <ChapterTree ref="chapterTreeRef" v-model:treeData="treeData" :level-names="['级', '章', '节', '目', '条', '款']"
        @action="handleAction">
        <template #node-actions="{ data, actions }">
          <!-- 内置按钮（必须手动保留） -->
          <el-button size="small" link type="info" @click.stop="actions.close(data)">关闭</el-button>
          <el-button size="small" link type="primary" @click.stop="actions.edit(data)">编辑</el-button>
          <el-button size="small" link type="primary" @click.stop="actions.add(data)">添加节</el-button>
          <el-button size="small" link type="danger" @click.stop="actions.delete(data)">删除</el-button>

          <!-- 自定义按钮（追加） -->
          <el-button size="small" link @click.stop="handleShare(data)">🔗 分享</el-button>
          <el-button size="small" link @click.stop="handleManage(data)">📚 题目管理</el-button>
        </template>
      </ChapterTree>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import ChapterTree, { type TreeNode, type TreeAction } from './components/ChapterTree.vue'

const chapterTreeRef = ref<InstanceType<typeof ChapterTree> | null>(null)
const treeData = ref<TreeNode[]>([
  {
    id: 1,
    label: '马克思主义原理',
    children: [
      {
        id: 11,
        label: '第一章：绪论',
        children: [
          { id: 111, label: '第一节：马克思主义的创立与发展', children: [] },
          { id: 112, label: '第二节：马克思主义的鲜明特征', children: [] },
          { id: 113, label: '第三节：马克思主义的当代价值', children: [] }
        ]
      },
      {
        id: 12,
        label: '第二章：世界的物质性及发展规律',
        children: [
          { id: 121, label: '第一节：物质与意识', children: [] },
          { id: 122, label: '第二节：事物的普遍联系和变化发展', children: [] }
        ]
      },
      {
        id: 13,
        label: '第三章：实践与认识及其发展规律',
        children: []
      }
    ]
  },
  {
    id: 2,
    label: '毛泽东思想和中国特色社会主义理论体系概论',
    children: [
      {
        id: 21,
        label: '第一章：毛泽东思想及其历史地位',
        children: [
          { id: 211, label: '第一节：毛泽东思想的形成和发展', children: [] },
          { id: 212, label: '第二节：毛泽东思想的主要内容和活的灵魂', children: [] }
        ]
      },
      {
        id: 22,
        label: '第二章：新民主主义革命理论',
        children: []
      }
    ]
  }
])
const loading = ref(false)
const lastLoadTime = ref('')

interface ActionLog {
  time: string
  action: string
  message: string
}

const actionLogs = ref<ActionLog[]>([])

const addLog = (action: TreeAction): void => {
  let message = ''

  switch (action.action) {
    case 'add':
      message = `添加节点 "${action.data.label}"` + (action.extra?.parentId ? ` (父级ID: ${action.extra.parentId})` : '')
      break
    case 'delete':
      message = `删除节点 "${action.data.label}"`
      break
    case 'update':
      message = `更新节点 "${action.data.label}"` + (action.data._oldLabel ? ` (原名称: ${action.data._oldLabel})` : '')
      break
    case 'drag':
      message = `拖拽节点到位置 ${action.extra?.position}` + (action.extra?.targetId ? ` (目标ID: ${action.extra.targetId})` : '')
      break
    case 'toggle':
      message = `${action.extra?.disabled ? '禁用' : '启用'}节点 "${action.data.label}"`
      break
    case 'batchEdit':
      message = `批量修改 ${action.extra?.items?.length || 0} 个节点`
      break
    default:
      message = `未知操作: ${action.action}`
  }

  actionLogs.value.unshift({
    time: new Date().toLocaleTimeString(),
    action: action.action,
    message
  })

  if (actionLogs.value.length > 20) {
    actionLogs.value.pop()
  }
}

const getLogClass = (action: string): string => {
  const classMap: Record<string, string> = {
    add: 'bg-green-50 text-green-700',
    delete: 'bg-red-50 text-red-700',
    update: 'bg-blue-50 text-blue-700',
    drag: 'bg-yellow-50 text-yellow-700',
    toggle: 'bg-purple-50 text-purple-700',
    batchEdit: 'bg-indigo-50 text-indigo-700'
  }
  return classMap[action] || 'bg-gray-50 text-gray-700'
}

// ========== API 数据类型定义 ==========
interface ApiTreeNode {
  id: number
  name: string
  level: number
  parent_id: number
  sort_order: number
  is_visible: boolean
  bank_id: number
  children?: ApiTreeNode[]
}

interface ApiResponse {
  code: number
  msg: string
  data: ApiTreeNode[]
}

// ========== 数据转换函数 ==========
const transformApiToTreeData = (apiNodes: ApiTreeNode[]): TreeNode[] => {
  return apiNodes.map((node) => ({
    id: node.id,
    label: node.name,
    children: node.children ? transformApiToTreeData(node.children) : [],
    _original: {
      level: node.level,
      parent_id: node.parent_id,
      sort_order: node.sort_order,
      is_visible: node.is_visible,
      bank_id: node.bank_id
    }
  }))
}

// ========== 统一接口调用 ==========
const callApi = async (action: TreeAction): Promise<void> => {
  try {
    const apiUrl = '/api/tree/action'

    const payload = {
      action: action.action,
      id: action.data.id,
      label: action.data.label,
      ...action.extra
    }

    console.log('[统一接口请求]', payload)

    // 调用后端API
    const response = await fetch(apiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const result = await response.json()

    if (result.code === 200) {
      console.log('[操作成功]', result.msg)
    } else {
      console.error('[操作失败]', result.msg)
    }
  } catch (error) {
    console.log('[模拟] 统一接口调用:', { action: action.action, data: action.data, extra: action.extra })
  }
}

// ========== 统一事件处理 ==========
const handleAction = async (action: TreeAction): Promise<void> => {
  addLog(action)
  await callApi(action)
}

// ========== 自定义按钮处理 ==========
const handleShare = (data: TreeNode): void => {
  console.log('分享节点:', data.label, data.id)
}

const handleManage = (data: TreeNode): void => {
  console.log('题目管理:', data.label, data.id)
}

// ========== 模拟 API 请求 ==========
const fetchTreeData = async (): Promise<void> => {
  loading.value = true

  try {
    const response = await fetch('/api/chapters')
    const result: ApiResponse = await response.json()

    if (result.code === 200) {
      treeData.value = transformApiToTreeData(result.data)
      lastLoadTime.value = new Date().toLocaleTimeString()
      console.log('数据加载成功:', treeData.value)
    } else {
      console.error('接口返回错误:', result.msg)
    }
  } catch (error) {
    console.log('使用模拟数据')

    const mockApiData: ApiResponse = {
      code: 200,
      msg: '操作成功',
      data: [
        {
          id: 1,
          name: "马克思主义原理",
          level: 1,
          parent_id: 0,
          sort_order: 0,
          is_visible: true,
          bank_id: 1,
          children: [
            {
              id: 2,
              name: "第一章 哲学概述",
              level: 2,
              parent_id: 1,
              sort_order: 0,
              is_visible: true,
              bank_id: 1,
              children: [
                {
                  id: 3,
                  name: "1.1 唯物主义",
                  level: 3,
                  parent_id: 2,
                  sort_order: 0,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                },
                {
                  id: 4,
                  name: "1.2 辩证法",
                  level: 3,
                  parent_id: 2,
                  sort_order: 1,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                }
              ]
            },
            {
              id: 5,
              name: "第二章 唯物史观",
              level: 2,
              parent_id: 1,
              sort_order: 1,
              is_visible: true,
              bank_id: 1,
              children: [
                {
                  id: 6,
                  name: "2.1 生产力与生产关系",
                  level: 3,
                  parent_id: 5,
                  sort_order: 0,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                }
              ]
            }
          ]
        },
        {
          id: 7,
          name: "习近平新时代中国特色社会主义思想",
          level: 1,
          parent_id: 0,
          sort_order: 1,
          is_visible: true,
          bank_id: 1,
          children: [
            {
              id: 8,
              name: "第一章 历史方位",
              level: 2,
              parent_id: 7,
              sort_order: 0,
              is_visible: true,
              bank_id: 1,
              children: [
                {
                  id: 9,
                  name: "1.1 新时代内涵",
                  level: 3,
                  parent_id: 8,
                  sort_order: 0,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                },
                {
                  id: 10,
                  name: "1.2 社会主要矛盾",
                  level: 3,
                  parent_id: 8,
                  sort_order: 1,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                }
              ]
            },
            {
              id: 11,
              name: "第二章 中心任务",
              level: 2,
              parent_id: 7,
              sort_order: 1,
              is_visible: true,
              bank_id: 1,
              children: [
                {
                  id: 12,
                  name: "2.1 中国式现代化",
                  level: 3,
                  parent_id: 11,
                  sort_order: 0,
                  is_visible: true,
                  bank_id: 1,
                  children: [
                    {
                      id: 13,
                      name: "2.1.1 现代化特征",
                      level: 4,
                      parent_id: 12,
                      sort_order: 0,
                      is_visible: true,
                      bank_id: 1,
                      children: []
                    }
                  ]
                }
              ]
            }
          ]
        },
        {
          id: 14,
          name: "思想道德与法治",
          level: 1,
          parent_id: 0,
          sort_order: 2,
          is_visible: true,
          bank_id: 1,
          children: [
            {
              id: 15,
              name: "第一章 人生价值",
              level: 2,
              parent_id: 14,
              sort_order: 0,
              is_visible: true,
              bank_id: 1,
              children: []
            },
            {
              id: 16,
              name: "第二章 道德修养",
              level: 2,
              parent_id: 14,
              sort_order: 1,
              is_visible: true,
              bank_id: 1,
              children: [
                {
                  id: 17,
                  name: "2.1 社会公德",
                  level: 3,
                  parent_id: 16,
                  sort_order: 0,
                  is_visible: true,
                  bank_id: 1,
                  children: []
                }
              ]
            }
          ]
        },
        {
          id: 18,
          name: "中国近现代史纲要",
          level: 1,
          parent_id: 0,
          sort_order: 3,
          is_visible: true,
          bank_id: 1,
          children: [
            {
              id: 19,
              name: "第一章 近代中国",
              level: 2,
              parent_id: 18,
              sort_order: 0,
              is_visible: true,
              bank_id: 1,
              children: []
            },
            {
              id: 20,
              name: "第二章 新民主主义革命",
              level: 2,
              parent_id: 18,
              sort_order: 1,
              is_visible: true,
              bank_id: 1,
              children: []
            }
          ]
        }
      ]
    }

    treeData.value = transformApiToTreeData(mockApiData.data)
    lastLoadTime.value = new Date().toLocaleTimeString()
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  // fetchTreeData()
})
</script>
