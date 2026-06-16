# 邮件模板编辑器 - 拖拽排序完整实现机制

## 一、架构总览

编辑器采用模块化设计，核心模块之间的协作关系如下：

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│       App       │────▶│  LayoutManager   │────▶│   BlockListManager  │
│  (事件绑定层)    │     │   (状态管理层)    │     │   (数据操作层)       │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
          │                                                          │
          │                     render                               │
          ▼                                                          ▼
┌─────────────────┐                                         ┌─────────────────┐
│ PreviewRenderer │                                         │ ComponentLibrary│
│  (预览渲染层)    │                                         │  (组件工厂)      │
└─────────────────┘                                         └─────────────────┘
```

### 全局拖拽状态变量（定义在 `app.js:3-7`）

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `draggingType` | string/null | 从组件库拖拽时的组件类型（如 'heading'、'paragraph'），内部区块拖拽时为 null |
| `draggingBlockId` | string/null | 正在拖拽的区块 ID，从组件库拖拽时为 null |
| `draggingParentBlockId` | string/null | 若拖拽的是 column 内的子组件，此为父级 columns 区块 ID；顶层拖拽为 null |
| `draggingColId` | string/null | 若拖拽的是 column 内的子组件，此为所属 column 的 ID；顶层拖拽为 null |
| `dragOverTarget` | object/null | 当前悬停目标的元数据，包含 `type`、`position`、`blockId`、`parentBlockId`、`colId` 等字段 |

---

## 二、三种典型拖拽场景详解

### 场景 1：从左侧组件库拖拽新组件（Copy 操作）

#### 事件绑定位置：`app.js:90-103`（`renderComponentList` 函数内）

```javascript
// .component-item 的 dragstart
item.addEventListener('dragstart', function(e) {
    draggingType = item.dataset.type;    // ✅ 设置组件类型
    draggingBlockId = null;              // ❌ 非区块拖拽
    draggingParentBlockId = null;
    draggingColId = null;
    e.dataTransfer.effectAllowed = 'copy';  // 标记为 copy 操作
    e.dataTransfer.setData('text/plain', draggingType);
});
```

#### dragover 计算插入位置

有两个层级可接收此类拖拽：

**A. 顶层 `editor-canvas` 空白区域** — `app.js:155-160`
- 不计算具体位置，drop 时默认追加到末尾

**B. 已有 `.block-wrapper` 上/下方插入** — `app.js:313-328`
```javascript
wrapper.addEventListener('dragover', function(e) {
    if (!draggingBlockId && !draggingType) return;
    if (draggingParentBlockId) return;  // 子组件拖拽不会触发顶层插入
    e.preventDefault();
    const rect = wrapper.getBoundingClientRect();
    const midPoint = rect.top + rect.height / 2;
    const position = e.clientY < midPoint ? 'top' : 'bottom';  // ⭐ 二分法判定
    setDragOver(wrapper, position, { type: 'main', blockId, position });
});
```

#### drop 时调用 LayoutManager 方法

**A. 落在 canvas 空白处** — `app.js:168-178`
```javascript
if (draggingType && !draggingBlockId) {
    const block = ComponentLibrary.createBlock(draggingType);
    LayoutManager.addBlock(block);  // 追加到末尾
}
```

**B. 落在某个 block-wrapper 上/下** — `app.js:336-353`
```javascript
const targetIndex = LayoutManager.getBlockIndex(blockId);
const pos = dragOverTarget && dragOverTarget.position;
const insertIndex = pos === 'top' ? targetIndex : targetIndex + 1;

if (draggingType && !draggingBlockId) {
    const block = ComponentLibrary.createBlock(draggingType);
    LayoutManager.addBlock(block, insertIndex);  // 插入到指定索引
}
```

#### 数据流：ComponentLibrary.createBlock → LayoutManager.addBlock → BlockListManager.addBlock

---

### 场景 2：顶层区块上下拖动调整顺序（Move 操作）

#### 事件绑定位置：`app.js:296-311`（`bindBlockEvents` 内的 `.block-drag-handle`）

```javascript
handle.addEventListener('dragstart', function(e) {
    e.stopPropagation();
    draggingType = null;                    // ❌ 非组件库拖拽
    draggingBlockId = blockId;              // ✅ 设置被拖拽的区块 ID
    draggingParentBlockId = null;           // 顶层区块无父级
    draggingColId = null;
    wrapper.classList.add('dragging');
    e.dataTransfer.effectAllowed = 'move';  // 标记为 move 操作
});
```

#### dragover 计算插入位置

与场景 1 共用相同逻辑 — `app.js:313-328`，通过二分法判断 `top`/`bottom`。

关键权限检查：
```javascript
if (draggingParentBlockId) return;  // ✅ 子组件不能在顶层 wrapper 上触发 dragover
```

#### drop 时调用 LayoutManager.moveBlock — `app.js:348-350`

```javascript
} else if (draggingBlockId && !draggingParentBlockId && draggingBlockId !== blockId) {
    const fromIndex = LayoutManager.getBlockIndex(draggingBlockId);
    LayoutManager.moveBlock(fromIndex, insertIndex);
}
```

**注意 `draggingBlockId !== blockId` 的判断**：防止把区块拖到自己身上导致无意义操作。

#### BlockListManager.moveBlock 的实现细节 — `blockList.js:50-57`

```javascript
function moveBlock(fromIndex, toIndex) {
    if (fromIndex === toIndex) return;
    const newBlocks = [...blocks];
    const [removed] = newBlocks.splice(fromIndex, 1);
    // ⭐ 关键修正：先删除后，目标索引若大于原索引需 -1
    const insertIndex = toIndex > fromIndex ? toIndex - 1 : toIndex;
    newBlocks.splice(insertIndex, 0, removed);
    setBlocks(newBlocks);
}
```

---

### 场景 3：Column 多栏布局内的子组件拖拽

此场景细分为两种子场景：**栏内 reorder** 和 **拖入新组件**。

#### 3.1 事件绑定位置

| 绑定对象 | 函数 | 行号 |
|----------|------|------|
| `.child-block-drag-handle` dragstart | `bindChildBlockEvents` | `app.js:451-460` |
| `.child-block-wrapper` dragover/drop | `bindChildBlockEvents` | `app.js:468-511` |
| `.column-content` dragover/drop | `bindColumnEvents` | `app.js:399-439` |

#### 3.2 子组件 dragstart — `app.js:451-460`

```javascript
handle.addEventListener('dragstart', function(e) {
    e.stopPropagation();
    draggingType = null;
    draggingBlockId = blockId;              // ✅ 子组件 ID
    draggingParentBlockId = parentBlockId;  // ✅ 父级 columns 区块 ID（关键标识）
    draggingColId = colId;                  // ✅ 所属 column ID
    wrapper.classList.add('dragging');
    e.dataTransfer.effectAllowed = 'move';
});
```

**`draggingParentBlockId` 和 `draggingColId` 是区分"顶层/子组件"拖拽的核心标识。**

#### 3.3 child-block-wrapper 的 dragover 权限控制 — `app.js:468-486`

```javascript
wrapper.addEventListener('dragover', function(e) {
    if (!draggingType && !draggingBlockId) return;
    // ⭐⭐⭐ 关键权限校验 ⭐⭐⭐
    if (draggingParentBlockId && draggingParentBlockId !== parentBlockId) return;
    if (draggingParentBlockId && draggingColId !== colId) return;
    // ... 通过校验后才计算 top/bottom 位置
});
```

**为什么要判断 `draggingParentBlockId !== parentBlockId`？**

这是一个**跨栏拖拽的权限拦截机制**，设计意图如下：

1. **防止数据结构混乱**：每个 column 内的 `blocks` 数组是独立维护的（参见 `LayoutManager.getColumnManager`，每次操作都创建独立的 `BlockListManager` 实例）。如果允许 A 栏的子组件直接拖到 B 栏的某个 `child-block-wrapper` 上方/下方，会出现「从 A 栏的 manager 删除，但却调用 B 栏的 moveBlock」的不一致问题。

2. **简化 drop 逻辑**：跨栏移动只允许通过 `.column-content`（栏的空白区域）作为 drop 目标（参见 `app.js:429-436`），此时走的是「深拷贝 + 删除原块 + 添加到新栏」的完整流程，避免了 moveBlock 在不同 manager 间的状态同步问题。

3. **UX 清晰性**：栏内 reorder 和跨栏移动是两种不同语义的操作，通过 dragover 阶段就拦截跨栏 hover，让用户从视觉反馈上就能区分「这里不能直接插入」。

#### 3.4 drop 分支逻辑

**A. 拖入新组件到栏内（copy）** — `app.js:503-505`
```javascript
if (draggingType && !draggingBlockId) {
    const block = ComponentLibrary.createBlock(draggingType);
    LayoutManager.addBlockToColumn(parentBlockId, colId, block, insertIndex);
}
```

**B. 栏内 reorder（move）** — `app.js:506-508`
```javascript
} else if (draggingBlockId && draggingParentBlockId === parentBlockId 
           && draggingColId === colId && draggingBlockId !== blockId) {
    const fromIndex = LayoutManager.getColumnBlockIndex(parentBlockId, colId, draggingBlockId);
    LayoutManager.moveBlockInColumn(parentBlockId, colId, fromIndex, insertIndex);
}
```

**C. 顶层区块拖入 column（移动+转换）** — `app.js:429-436`（绑定在 `.column-content` 上）
```javascript
} else if (draggingBlockId && !draggingParentBlockId) {
    const block = LayoutManager.getState().blocks.find(b => b.id === draggingBlockId);
    if (block && block.type !== 'columns') {  // columns 类型不允许嵌套
        const newBlock = JSON.parse(JSON.stringify(block));
        newBlock.id = 'block_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
        LayoutManager.addBlockToColumn(parentBlockId, colId, newBlock);
        LayoutManager.removeBlock(draggingBlockId);
    }
}
```

---

## 三、拖拽各阶段的完整状态流转

### 3.1 dragstart 阶段：状态初始化

| 拖拽来源 | draggingType | draggingBlockId | draggingParentBlockId | draggingColId | effectAllowed |
|----------|-------------|-----------------|----------------------|---------------|---------------|
| 组件库 `.component-item` | 组件类型值 | null | null | null | `copy` |
| 顶层 `.block-drag-handle` | null | 区块 ID | null | null | `move` |
| 栏内 `.child-block-drag-handle` | null | 子组件 ID | 父 columns ID | column ID | `move` |

### 3.2 dragover 阶段：位置计算 + 权限校验

```
用户移动鼠标 → dragover 事件触发
    │
    ├─ 第一步：快速过滤（draggingType 和 draggingBlockId 都为空则直接 return）
    │
    ├─ 第二步：权限校验（根据绑定元素不同有不同规则）
    │   ├─ .block-wrapper: draggingParentBlockId 必须为空（只接收顶层拖拽）
    │   ├─ .child-block-wrapper: draggingParentBlockId 必须等于当前 parentBlockId
    │   │                       且 draggingColId 必须等于当前 colId（同栏校验）
    │   └─ .column-content: 无特殊限制，接收所有类型
    │
    └─ 第三步：计算插入位置
        ├─ block-wrapper / child-block-wrapper: 用二分法（鼠标 Y < 中点？top : bottom）
        └─ column-content: 固定 position = 'bottom'（追加到末尾）
```

`setDragOver` 函数 — `app.js:534-542`：
- 先清除所有旧的高亮样式
- 设置 `dragOverTarget` 全局变量保存目标元数据
- 给当前元素添加 `drag-over-top` 或 `drag-over-bottom` CSS 类

### 3.3 drop 阶段：数据变更

根据全局状态组合路由到不同的 LayoutManager 方法：

```
drop 事件触发
    │
    ├─ draggingType 有值（从组件库拖入）
    │   ├─ 目标是顶层 canvas/wrapper → LayoutManager.addBlock(block, index?)
    │   └─ 目标是 column-content/child-wrapper → LayoutManager.addBlockToColumn(...)
    │
    └─ draggingBlockId 有值（内部区块移动）
        ├─ 顶层移动（!draggingParentBlockId）
        │   └─ LayoutManager.moveBlock(fromIndex, toIndex)
        │
        ├─ 栏内同栏移动（draggingParentBlockId && 同 colId）
        │   └─ LayoutManager.moveBlockInColumn(...)
        │
        └─ 顶层 → column 移动（!draggingParentBlockId && 目标是 column-content）
            └─ 深拷贝 + LayoutManager.addBlockToColumn(...) + LayoutManager.removeBlock(...)
```

### 3.4 dragend 阶段：状态清理

所有拖拽源的 dragend 事件都会调用：

```javascript
function clearDragState() {  // app.js:551-556
    draggingType = null;
    draggingBlockId = null;
    draggingParentBlockId = null;
    draggingColId = null;
}

function clearDragOver() {   // app.js:544-549
    document.querySelectorAll('.drag-over-top, .drag-over-bottom, .drag-over')
        .forEach(el => el.classList.remove(...));
    dragOverTarget = null;
}
```

**为什么 dragend 和 drop 都要清理？**
- drop 只在成功放下时触发
- dragend 无论成功/取消（按 Esc、拖到窗口外）都会触发
- 双重保险避免状态残留

---

## 四、模块协作调用链

### 4.1 时序图：从鼠标按下到重新渲染

以「顶层区块 move」为例：

```
用户鼠标按下 .block-drag-handle
    │
    ▼
App.bindBlockEvents ── dragstart ──▶ 设置 draggingBlockId = xxx
    │                                   draggingParentBlockId = null
    │                                   effectAllowed = 'move'
    │
    ▼  用户拖动鼠标
App.bindBlockEvents ── dragover (在目标 .block-wrapper 上)
    │
    ├─ 权限校验：draggingParentBlockId 为空 → 通过
    ├─ 二分法计算 top/bottom
    └─ setDragOver() → 写入 dragOverTarget + 添加 CSS 高亮
    │
    ▼  用户松开鼠标
App.bindBlockEvents ── drop
    │
    ├─ 读取 dragOverTarget.position
    ├─ 计算 insertIndex
    ├─ LayoutManager.getBlockIndex(draggingBlockId) → fromIndex
    └─ LayoutManager.moveBlock(fromIndex, insertIndex)
        │
        ▼
    LayoutManager.onMainListChange() 回调
        │
        ├─ App.onStateChange(state)
        │   ├─ saveStateToStorage(state)       // 持久化到 localStorage
        │   ├─ App.renderEditor()              // 重新渲染编辑区 DOM
        │   │   └─ bindBlockEvents(canvas)     // ⚠️ 重新绑定所有事件（DOM 重建了）
        │   ├─ PreviewRenderer.render(blocks)  // 更新 iframe 预览
        │   └─ App.renderProperties()          // 刷新属性面板
        │
        └─ App.bindBlockEvents ── dragend
            ├─ wrapper.classList.remove('dragging')
            ├─ clearDragState()
            └─ clearDragOver()
```

### 4.2 LayoutManager 与 BlockListManager 的协作

```
┌───────────────────────────────────────────────────────────┐
│                     LayoutManager                          │
│                                                           │
│  mainList ──────────────────▶ BlockListManager (顶层)      │
│                                - blocks: [block1, block2]  │
│                                                           │
│  getColumnManager(blockId, colId)                          │
│    │                                                       │
│    └─▶ 找到 block.data.children[i]                         │
│         └─▶ BlockListManager (栏内临时实例)                 │
│              - blocks: col.blocks (深拷贝)                  │
│              - onChange: syncColumnBlocksToMain()          │
│                   └─ 将变更写回主列表                       │
└───────────────────────────────────────────────────────────┘
```

**关键点**：每个 column 的 blocks 不是直接操作的，而是通过 `getColumnManager()` 创建一个**临时的 BlockListManager 实例**，该实例的 onChange 回调会把变更同步回主 blocks 数组的对应位置。这实现了「栏内操作」和「顶层操作」使用完全相同的 BlockListManager API，避免代码重复。

---

## 五、完整流程图

### 5.1 drop 事件决策树

```
                           drop 事件
                              │
               ┌──────────────┴──────────────┐
               │                             │
      draggingType != null?          draggingBlockId != null?
          (组件库拖拽)                    (内部移动)
               │                             │
      ┌────────┴────────┐          ┌─────────┴──────────┐
      │                 │          │                    │
  顶层目标?         栏内目标?    顶层拖拽?          栏内拖拽?
 (canvas/wrapper) (column/child) (!parentId)      (parentId存在)
      │                 │          │                    │
      ▼                 ▼          ▼          ┌─────────┴─────────┐
addBlock()     addBlockToColumn() moveBlock()  │                   │
                                          同 parentId?        顶层→栏内?
                                          同 colId?           (!parentId &&
                                              │                目标是column)
                                              ▼                   │
                                    moveBlockInColumn()           ▼
                                                        深拷贝+addBlockToColumn
                                                        +removeBlock(原位置)
```

---

## 六、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 全局拖拽状态定义 | `js/app.js` | 3-7 |
| 组件库 dragstart | `js/app.js` | 91-98 |
| canvas dragover/drop | `js/app.js` | 155-178 |
| bindBlockEvents（顶层区块事件） | `js/app.js` | 291-380 |
| 顶层 dragstart | `js/app.js` | 296-305 |
| 顶层 dragover（top/bottom 计算） | `js/app.js` | 313-328 |
| 顶层 drop | `js/app.js` | 336-353 |
| bindColumnEvents | `js/app.js` | 382-442 |
| column-content drop | `js/app.js` | 418-439 |
| bindChildBlockEvents | `js/app.js` | 444-532 |
| 子组件 dragstart | `js/app.js` | 451-460 |
| 子组件 dragover（跨栏权限校验） | `js/app.js` | 468-486 |
| 子组件 drop | `js/app.js` | 494-511 |
| setDragOver / clearDragOver | `js/app.js` | 534-549 |
| clearDragState | `js/app.js` | 551-556 |
| LayoutManager.init | `js/layout.js` | 8-18 |
| LayoutManager.addBlock | `js/layout.js` | 60-62 |
| LayoutManager.moveBlock | `js/layout.js` | 71-73 |
| LayoutManager.getColumnManager | `js/layout.js` | 109-124 |
| LayoutManager.addBlockToColumn | `js/layout.js` | 126-130 |
| LayoutManager.moveBlockInColumn | `js/layout.js` | 143-147 |
| LayoutManager.syncColumnBlocksToMain | `js/layout.js` | 89-107 |
| BlockListManager.createManager | `js/blockList.js` | 3-128 |
| BlockListManager.moveBlock（索引修正） | `js/blockList.js` | 50-57 |
| ComponentLibrary.createBlock | `js/components.js` | 224-245 |
| PreviewRenderer.render | `js/preview.js` | 12-19 |
