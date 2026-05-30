# ChapterTree - 章节树管理组件

基于 **Vue 3 + Element Plus + Tailwind CSS** 的 el-tree 二次封装，实现章节/分类的增删改查、拖拽排序、批量操作等功能。

## ✨ 功能特性

- 🎯 **无限层级嵌套** - 支持多级章节结构
- 🔄 **拖拽排序** - 同级拖拽调整顺序，跨层级拖拽改变父节点
- ➕ **批量添加** - 支持一次性添加多个章节
- ✏️ **批量修改** - 批量修改选中节点及其子节点
- 🖱️ **双击编辑** - 双击节点文字快速编辑重命名
- 🔒 **启用/禁用** - 节点禁用状态管理
- 🎨 **层级颜色** - 不同层级用不同背景色区分
- 📡 **统一接口** - 所有操作通过 `@action` 事件统一抛出
- ⚙️ **高度可配置** - 自定义序号、层级名称等参数

## 📦 安装

```bash
# 克隆项目
git clone <your-repo-url>
cd el-tree-from

# 安装依赖
npm install
```

## 🚀 快速开始

### 1. 开发模式运行

```bash
npm run dev
```

访问 http://localhost:5173

### 2. 编译生产版本

```bash
# 编译
npm run build

# 预览编译结果
npm run preview

# 或使用静态服务器
npm run serve
```

## 📖 使用方法

### 基础用法

```vue
<template>
  <ChapterTree 
    v-model:treeData="treeData" 
    @action="handleAction" 
  />
</template>

<script setup lang="ts">
import { ref } from 'vue'
import ChapterTree, { type TreeNode, type TreeAction } from './components/ChapterTree.vue'

const treeData = ref<TreeNode[]>([
  {
    id: '1',
    label: '第一章',
    children: [
      {
        id: '1-1',
        label: '第一节',
        children: []
      }
    ]
  }
])

// 统一处理所有操作
const handleAction = (action: TreeAction) => {
  console.log('操作:', action.action, action.data, action.extra)
}
</script>
```

### 完整示例

完整示例代码请查看 [src/App.vue](src/App.vue)，包含：

- 🔄 **从远程API加载数据** - `fetchTreeData()` + 数据格式转换
- 📡 **统一事件处理** - `handleAction()` 处理所有增删改查操作
- 📝 **操作日志面板** - 实时显示所有操作记录
- ⚙️ **配置控制** - 序号显示开关、自定义层级名称
- 🔗 **后端接口对接** - 统一API调用函数

**快速查看核心代码：**

```vue
<!-- src/App.vue 核心用法 -->
<template>
  <ChapterTree 
    ref="treeRef"
    v-model:treeData="treeData"
    :show-index="showIndex"
    :level-names="['级', '章', '节', '目', '条', '款']"
    @action="handleAction"    <!-- 统一处理所有操作 -->
  />
</template>

<script setup lang="ts">
import ChapterTree, { type TreeNode, type TreeAction } from './components/ChapterTree.vue'

const treeRef = ref()
const treeData = ref<TreeNode[]>([])

// 所有操作统一处理
const handleAction = async (action: TreeAction) => {
  console.log('操作:', action.action, action.data, action.extra)
  
  // 调用后端统一接口
  await fetch('/api/tree/action', {
    method: 'POST',
    body: JSON.stringify({
      action: action.action,
      id: action.data.id,
      label: action.data.label,
      ...action.extra
    })
  })
}

// 从API获取数据（name -> label 转换）
const fetchTreeData = async () => {
  const res = await fetch('/api/chapters')
  const result = await res.json()
  treeData.value = transformApiToTree(result.data)
}

// API数据格式转换
const transformApiToTree = (nodes: any[]): TreeNode[] =>
  nodes.map(n => ({
    id: n.id,
    label: n.name,           // name → label
    children: n.children ? transformApiToTree(n.children) : []
  }))
</script>
```

## 📋 Props 参数

| 参数 | 类型 | 默认值 | 必填 | 说明 |
|------|------|--------|------|------|
| `treeData` | `TreeNode[]` | - | ✅ | 树形数据 |
| `showIndex` | `boolean` | `true` | ❌ | 是否显示序号前缀（第1章、第1节） |
| `levelNames` | `string[]` | `['章','节','小节','条','款']` | ❌ | 自定义各层级名称 |
| `actions` | `NodeActions \| false` | `{全部:true}` | ❌ | 控制内置按钮显隐，设为 `false` 隐藏全部 |

### 数据类型定义

```typescript
// 节点类型
interface TreeNode {
  id: string | number          // 节点唯一标识
  label: string                // 节点显示名称
  children?: TreeNode[]        // 子节点数组
  [key: string]: any           // 其他自定义字段
}

// 操作动作类型
interface TreeAction {
  action: 'add' | 'delete' | 'update' | 'drag' | 'copy' | 'toggle' | 'batchEdit'
  data: TreeNode               // 操作的节点数据
  extra?: {
    parentId?: string | number   // 添加时的父级ID
    targetId?: string | number   // 拖拽目标ID
    position?: string            // 拖拽位置: before / after / inner
    items?: Array<{              // 批量修改项
      id: string | number
      oldLabel: string
      newLabel: string
    }>
    disabled?: boolean           // 启用/禁用状态
  }
}
```

## 🎛️ 事件说明

### 统一事件 `@action`（推荐）

所有操作统一通过此事件抛出，包含完整的操作信息：

```vue
<ChapterTree @action="onAction" />

<script setup>
const onAction = (action: TreeAction) => {
  switch (action.action) {
    case 'add':
      // 添加节点
      console.log('新增:', action.data.label, '父级:', action.extra.parentId)
      break
      
    case 'delete':
      // 删除节点
      console.log('删除:', action.data.id)
      break
      
    case 'update':
      // 更新节点名称
      console.log('更新:', action.data._oldLabel, '->', action.data.label)
      break
      
    case 'drag':
      // 拖拽排序
      console.log('拖拽到:', action.extra.position, '目标:', action.extra.targetId)
      break
      
    case 'toggle':
      // 启用/禁用
      console.log(action.extra.disabled ? '已禁用' : '已启用')
      break
      
    case 'batchEdit':
      // 批量修改
      console.log('修改数量:', action.extra.items?.length)
      break
  }
}
</script>
```

### 单独事件（兼容旧版）

| 事件名 | 参数 | 触发时机 |
|--------|------|----------|
| `@add` | `(node, parentId?)` | 添加节点时 |
| `@delete` | `(node)` | 删除节点时 |
| `@update` | `(node)` | 编辑节点名称后 |
| `@drag` | `{from, to, position}` | 拖拽完成后 |
| `@copy` | `(node)` | 复制节点时 |
| `@select` | `(node)` | 点击选中节点时 |
| `@update:treeData` | `(data)` | 数据变化时 |

## 🔧 方法暴露

通过 `ref` 调用组件内部方法：

```vue
<template>
  <ChapterTree ref="treeRef" />
  <button @click="expandAll">展开全部</button>
  <button @click="collapseAll">折叠全部</button>
</template>

<script setup>
import { ref } from 'vue'
import ChapterTree from './components/ChapterTree.vue'

const treeRef = ref<InstanceType<typeof ChapterTree>>()

// 展开所有节点
const expandAll = () => {
  treeRef.value?.expandAll()
}

// 折叠所有节点
const collapseAll = () => {
  treeRef.value?.collapseAll()
}

// 获取当前树数据
const getData = () => {
  const data = treeRef.value?.getTreeData()
  console.log(data)
}

// 设置树数据
const setData = (newData: TreeNode[]) => {
  treeRef.value?.setTreeData(newData)
}
</script>
```

| 方法名 | 返回值 | 说明 |
|--------|--------|------|
| `getTreeData()` | `TreeNode[]` | 获取当前树数据 |
| `setTreeData(data)` | `void` | 设置树数据 |
| `expandAll()` | `void` | 展开所有节点 |
| `collapseAll()` | `void` | 折叠所有节点 |
| `handleClose(data)` | `void` | 关闭/启用节点（外部调用） |
| `handleShare(data)` | `void` | 分享节点（外部调用） |
| `handleEditNode(data)` | `void` | 编辑节点（外部调用） |
| `handleAddSubNode(data)` | `void` | 添加子节点（外部调用） |
| `handleManage(data)` | `void` | 题目管理（外部调用） |
| `handleDeleteNode(data)` | `void` | 删除节点（外部调用） |

## 🎨 自定义配置

### 隐藏序号前缀

```vue
<!-- 不显示 "第1章：" 前缀 -->
<ChapterTree :show-index="false" />
```

效果：
- `showIndex=true`: 第1章：基础知识
- `showIndex=false`: 基础知识

### 自定义层级名称

```vue
<!-- 自定义各层级的称呼 -->
<ChapterTree :level-names="['卷', '篇', '章', '节', '段']" />
```

效果：

| 层级 | 默认 | 自定义 |
|------|------|--------|
| 第1层 | 第1章 | **第1卷** |
| 第2层 | 第1节 | **第1篇** |
| 第3层 | 第1小节 | **第1章** |
| 第4层 | 第1条 | **第1节** |
| 第5层 | 第1款 | **第1段** |

### 控制内置按钮显隐

通过 `actions` 参数控制每个按钮的显示/隐藏：

```vue
<!-- 默认显示：关闭/启用、编辑、添加节、删除 -->
<!-- 需要显示"分享"或"题目管理"时手动开启 -->
<ChapterTree :actions="{ share: true, manage: true }" />

<!-- 只显示编辑和删除 -->
<ChapterTree :actions="{ edit: true, delete: true, close: false, add: false }" />

<!-- 隐藏所有内置按钮（完全自定义） -->
<ChapterTree :actions="false" />
```

**可用配置项：**

| 配置键 | 对应按钮 | 默认值 | 说明 |
|--------|----------|--------|------|
| `close` | 关闭/启用 | ✅ `true` | 内置 |
| `edit` | 编辑 | ✅ `true` | 内置 |
| `add` | 添加节 | ✅ `true` | 内置 |
| `delete` | 删除 | ✅ `true` | 内置 |
| `share` | 分享 | ❌ `false` | **自定义**（需插槽） |
| `manage` | 题目管理 | ❌ `false` | **自定义**（需插槽） |

> 💡 **提示**：`close/edit/add/delete` 是核心操作，默认显示。  
> `share/manage` 属于业务特定功能，默认隐藏，通过插槽自定义实现。

### 自定义操作按钮（插槽）

使用 `#node-actions` 插槽添加自定义按钮（如"分享"、"题目管理"等业务功能）：

```vue
<template>
  <!-- 默认显示：关闭/启用、编辑、添加节、删除 -->
  <!-- 通过插槽追加自定义按钮 -->
  <ChapterTree @action="handleAction">
    <template #node-actions="{ data, actions }">
      <!-- ⚠️ 内置按钮（使用插槽时必须手动保留） -->
      <el-button size="small" link type="info" @click.stop="actions.close(data)">关闭</el-button>
      <el-button size="small" link type="primary" @click.stop="actions.edit(data)">编辑</el-button>
      <el-button size="small" link type="primary" @click.stop="actions.add(data)">添加节</el-button>
      <el-button size="small" link type="danger" @click.stop="actions.delete(data)">删除</el-button>

      <!-- ✅ 追加自定义按钮 -->
      <el-button size="small" link @click.stop="handleShare(data)">🔗 分享</el-button>
      <el-button size="small" link @click.stop="handleManage(data)">📚 题目管理</el-button>
      <el-button size="small" link @click.stop="handleExport(data)">📤 导出</el-button>
    </template>
  </ChapterTree>
</template>

<script setup lang="ts">
import ChapterTree from './components/ChapterTree.vue'
import type { TreeNode } from './components/ChapterTree.vue'

// 自定义操作处理
const handleShare = (data: TreeNode) => {
  console.log('分享:', data.label)
}

const handleManage = (data: TreeNode) => {
  console.log('题目管理:', data.id)
  // 跳转到题目管理页面
  // router.push(`/questions?chapterId=${data.id}`)
}

const handleExport = (data: TreeNode) => {
  console.log('导出:', data.label)
}
</script>
```

> ⚠️ **重要提示**：当使用 `#node-actions` 插槽时，**插槽内容会完全替换默认按钮**！  
> 如果需要保留内置的4个核心按钮（关闭、编辑、添加节、删除），必须如上例所示在插槽中**手动调用 `actions.xxx()` 方法**。

**不使用插槽（推荐简单场景）：**

```vue
<!-- 自动显示内置4个按钮，无需写任何插槽 -->
<ChapterTree v-model:treeData="data" />
```

**插槽作用域参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `node` | `ElTreeNode` | el-tree 节点对象（含 level、isLeaf 等） |
| `data` | `TreeNode` | 节点数据（id、label 等） |
| `actions` | `object` | 内置方法对象（可调用内置方法） |

**actions 方法列表（可在插槽中调用）：**

```typescript
{
  close: (data: TreeNode) => void     // 关闭/启用
  share: (data: TreeNode) => void     // 分享
  edit: (data: TreeNode) => void      // 编辑
  add: (data: TreeNode) => void       // 添加子节点
  manage: (data: TreeNode) => void    // 题目管理
  delete: (data: TreeNode) => void    // 删除
}
```

## 📡 后端接口对接

### 数据格式要求

#### 请求树数据

```
GET /api/chapters

Response:
{
  "code": 200,
  "msg": "success",
  "data": [
    {
      "id": 3,
      "name": "马原",           // ← 对应 label
      "level": 1,
      "parent_id": 0,
      "sort_order": 0,
      "is_visible": true,
      "bank_id": 1,
      "children": [...]
    }
  ]
}
```

> 注意：API返回的 `name` 字段会自动映射为组件所需的 `label`

#### 统一操作接口

```
POST /api/tree/action

Body:
{
  "action": "add",                    // 操作类型
  "id": 123,                          // 节点ID
  "label": "新章节名称",              // 节点名称
  "parentId": 3                       // 额外参数（根据action变化）
}
```

**不同操作的请求参数：**

| Action | 必需参数 | 可选参数 |
|--------|----------|----------|
| `add` | id, label | parentId |
| `delete` | id | - |
| `update` | id, label | _oldLabel |
| `drag` | id | targetId, position |
| `toggle` | id, disabled | - |
| `batchEdit` | data | items[] |

### 数据转换函数

```typescript
// API数据 → 组件数据
function transformApiToTree(apiNodes: ApiNode[]): TreeNode[] {
  return apiNodes.map(node => ({
    id: node.id,
    label: node.name,              // name → label 映射
    children: node.children 
      ? transformApiToTree(node.children) 
      : [],
    _original: {                   // 保留原始字段
      level: node.level,
      parent_id: node.parent_id,
      sort_order: node.sort_order,
      is_visible: node.is_visible
    }
  }))
}

// 组件数据 → API数据
function transformTreeToApi(treeNodes: TreeNode[]): ApiNode[] {
  return treeNodes.map(node => ({
    id: node.id,
    name: node.label,              // label → name 映射
    children: node.children 
      ? transformTreeToApi(node.children) 
      : [],
    level: node._original?.level,
    parent_id: node._original?.parent_id,
    sort_order: node._original?.sort_order,
    is_visible: node._original?.is_visible ?? true
  }))
}
```

## 🖥️ 操作指南

### 添加章节

1. **顶部按钮**：点击「添加分类」→ 弹出对话框 → 输入名称 → 点击「添加」继续添加或「确定」提交
2. **节点内添加**：点击某节点的「添加节」→ 自动以该节点为父级 → 弹出对话框添加子节点
3. **批量添加**：在对话框中连续输入多个名称，一次性全部提交

### 编辑节点

- **方式1**：双击节点文字 → 进入编辑模式 → 输入新名称 → 按 Enter 或点击外部保存
- **方式2**：点击节点的「编辑」按钮 → 同上

### 删除节点

点击节点的「删除」按钮 → 弹出确认框 → 确认后删除该节点及所有子节点

### 启用/禁用

点击节点的「关闭」/「启用」按钮切换状态：
- 已禁用的节点显示为半透明+灰色背景+删除线
- 禁用时其他操作按钮不可点击

### 拖拽排序

- **同级拖拽**：上下拖动调整顺序（自动重新计算序号）
- **跨层级拖拽**：左右拖拽改变父节点
- **视觉反馈**：拖拽元素显示虚线框，放置位置有高亮提示

### 批量修改

1. 先选中一个节点（作为修改范围）
2. 点击顶部「批量修改」按钮
3. 在弹出的对话框中修改该节点及其所有子节点的名称
4. 可勾选/取消勾选需要修改的项
5. 点击「确定修改」批量提交

## 🎯 使用场景

- 📚 **课程章节管理** - 教材目录、课程大纲
- 📁 **分类目录管理** - 商品分类、文档分类
- 🏗️ **项目结构管理** - 项目模块、任务分解
- 🗂️ **知识库管理** - 知识点层级、FAQ分类
- 📊 **组织架构管理** - 部门层级、人员分组

## 🛠️ 技术栈

- Vue 3 (Composition API)
- Element Plus (el-tree)
- TypeScript
- Tailwind CSS
- Vite 6

## 📁 项目结构

```
el-tree-from/
├── src/
│   ├── components/
│   │   └── ChapterTree.vue     # 主组件
│   ├── App.vue                 # 示例父组件
│   ├── main.ts                 # 入口文件
│   └── style.css               # 全局样式
├── package.json
├── vite.config.ts
└── README.md
```

## 📄 License

MIT License

---

## 💡 常见问题

### Q: 如何对接后端API？

A: 监听 `@action` 事件，将操作发送到后端统一接口。参考「后端接口对接」章节。

### Q: 序号前缀可以隐藏吗？

A: 可以，设置 `:show-index="false"` 即可。

### Q: 可以自定义层级名称吗？

A: 可以，通过 `:level-names="['卷','篇','章']"` 传入自定义名称数组。

### Q: 如何获取当前树的所有数据？

A: 通过 ref 调用 `getTreeData()` 方法。

### Q: 如何隐藏"分享"、"题目管理"等不需要的按钮？

A: 默认已隐藏！`share` 和 `manage` 默认不显示。如需显示：
```vue
<ChapterTree :actions="{ share: true, manage: true }" />
```

### Q: 如何添加自定义按钮（如分享、题目管理）？

A: 使用 `#node-actions` 插槽（**注意：必须手动保留内置按钮**）：
```vue
<ChapterTree>
  <template #node-actions="{ data, actions }">
    <!-- 必须手动保留内置按钮 -->
    <el-button size="small" link type="info" @click="actions.close(data)">关闭</el-button>
    <el-button size="small" link type="primary" @click="actions.edit(data)">编辑</el-button>
    <el-button size="small" link type="primary" @click="actions.add(data)">添加节</el-button>
    <el-button size="small" link type="danger" @click="actions.delete(data)">删除</el-button>

    <!-- 追加自定义按钮 -->
    <el-button size="small" link @click="handleShare(data)">🔗 分享</el-button>
    <el-button size="small" link @click="handleManage(data)">📚 题目管理</el-button>
  </template>
</ChapterTree>
```

### Q: 为什么使用插槽后内置按钮消失了？

A: 这是 Vue 插槽的正常行为 - **插槽内容会完全替换默认内容**。  
解决方法：在插槽中通过 `actions.xxx(data)` 手动调用内置方法。

### Q: 编译后如何部署？

A: 运行 `npm run build` 生成 `dist` 目录，将其部署到任意静态服务器即可。
