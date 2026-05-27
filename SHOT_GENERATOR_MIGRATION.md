# Shot Generator 移植设计文档

> 目标：将 Storyboarder 的 Shot Generator 功能移植到 FlowCanvas 项目，作为无限画布上的一个独立节点。

---

## 目录

1. [原始代码架构分析](#1-原始代码架构分析)
2. [目标架构设计](#2-目标架构设计)
3. [文件结构](#3-文件结构)
4. [核心实现：Zustand Store](#4-核心实现zustand-store)
5. [核心实现：Konva 节点](#5-核心实现konva-节点)
6. [核心实现：编辑器 Overlay](#6-核心实现编辑器-overlay)
7. [核心实现：R3F 场景管理器](#7-核心实现r3f-场景管理器)
8. [核心实现：Three.js 组件移植](#8-核心实现threejs-组件移植)
9. [核心实现：IK 系统适配](#9-核心实现ik-系统适配)
10. [核心实现：资源加载系统](#10-核心实现资源加载系统)
11. [核心实现：保存与持久化](#11-核心实现保存与持久化)
12. [后端接口](#12-后端接口)
13. [R3F v4 → v8 迁移手册](#13-r3f-v4--v8-迁移手册)
14. [执行计划](#14-执行计划)

---

## 1. 原始代码架构分析

### 1.1 进程结构

Shot Generator 原本是一个独立的 Electron `BrowserWindow`，通过 IPC 与主应用通信。

```
Storyboarder 主窗口
    │
    ├── ipcRenderer.send('shot-generator:open')
    │
Shot Generator BrowserWindow (独立窗口)
    ├── window.js            ← 渲染进程初始化，创建 Redux store
    ├── Editor 组件          ← React 根组件
    │   ├── SceneManagerR3fLarge   ← 主相机视图 (透视投影)
    │   ├── SceneManagerR3fSmall   ← 俯视图 (正交投影)
    │   ├── Toolbar
    │   ├── ElementsPanel
    │   └── InspectorPanel
    └── service.js           ← IPC 通信代理
```

### 1.2 核心文件清单

| 路径 | 行数 | 职责 |
|------|------|------|
| `src/js/windows/shot-generator/window.js` | ~200 | 渲染进程初始化，Redux store 配置，IPC 监听 |
| `src/js/windows/shot-generator/service.js` | ~80 | IPC 通信代理（saveShot、insertShot、getBoard） |
| `src/js/windows/shot-generator/main.js` | ~150 | 主进程窗口管理 |
| `src/js/shot-generator/components/Editor/index.js` | 302 | React 根组件，布局，截图触发 |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | ~600 | 主相机视图场景管理，connect Redux |
| `src/js/shot-generator/SceneManagerR3fSmall.js` | ~300 | 俯视图场景管理，connect Redux |
| `src/js/shot-generator/CameraUpdate.js` | ~60 | 同步 Redux camera 属性到 Three.js 相机 |
| `src/js/shared/reducers/shot-generator.js` | ~1500 | Redux reducer，所有 actions，selectors |
| `src/js/shared/IK/SGIkHelper.js` | ~350 | IK 系统单例，管理 Ragdoll 和控制点 |
| `src/js/shared/IK/objects/IkObjects/Ragdoll.js` | ~400 | IK 链条求解器 |

### 1.3 Redux State 完整结构

```typescript
// state.undoable.present 部分（受撤销/重做保护）
{
  sceneObjects: {
    [id: string]: {
      id: string
      type: 'character' | 'object' | 'light' | 'camera' | 'volume' | 'image' | 'group'
      x: number; y: number; z: number
      rotation: number       // Y轴旋转（弧度）
      visible: boolean
      locked: boolean
      blocked: boolean       // 多人协作时被他人锁定

      // type === 'character' 专属
      model: string          // 'adult-male' | 'adult-female' | 自定义路径
      height: number
      headScale: number
      tintColor: string
      morphTargets: { mesomorphic: number; ectomorphic: number; endomorphic: number }
      skeleton: Record<boneName, { rotation: { x: number; y: number; z: number } }>
      handSkeleton: Record<boneName, { rotation: { x: number; y: number; z: number } }>
      poleTargets: Record<string, { position: Vec3; rotation: Vec3 }>
      ragDoll: boolean       // 是否启用 IK
      characterPresetId?: string
      posePresetId?: string
      handPosePresetId?: string
      emotionPresetId?: string

      // type === 'object' 专属
      model: string          // 'box' | GLTF 路径
      width: number; height: number; depth: number
      tintColor: string

      // type === 'light' 专属
      intensity: number
      angle: number
      distance: number
      penumbra: number
      decay: number
      tilt: number
      roll: number

      // type === 'camera' 专属
      fov: number
      roll: number

      // type === 'attachable' 专属
      attachToId: string     // 绑定到哪个角色 ID
      attachBoneName: string // 绑定到哪根骨骼

      // type === 'group' 专属
      children: string[]     // 子对象 ID 列表
    }
  }

  world: {
    ground: boolean
    backgroundColor: string
    room: { visible: boolean; width: number; length: number; height: number }
    environment: { file: string; x: number; y: number; z: number; rotation: number; scale: number; visible: boolean; grayscale: boolean }
    ambient: { intensity: number }
    directional: { intensity: number; rotation: number; tilt: number }
    fog: { visible: boolean; far: number }
    shadingMode: 'outline' | 'solid' | 'flat' | 'grayscale'
  }

  selections: string[]
  selectedBone: { characterId: string; boneName: string } | null
  selectedAttachable: string | null
  activeCamera: string
}

// state 顶层（不受撤销保护）
{
  meta: { storyboarderFilePath: string; lastSavedHash: string }
  models: Record<string, ModelDescriptor>
  presets: {
    scenes: Record<string, ScenePreset>
    characters: Record<string, CharacterPreset>
    poses: Record<string, PosePreset>
    handPoses: Record<string, HandPosePreset>
    emotions: Record<string, EmotionPreset>
  }
  mainViewCamera: 'live' | 'ortho'
  aspectRatio: number
  board: BoardObject
  cameraShots: Record<cameraId, { size: string; angle: string; character: string }>
}
```

### 1.4 渲染管线

```
SceneManagerR3fLarge（主相机视图）
├── 场景对象
│   ├── Character          ← SkinnedMesh + LOD + 骨骼姿势 + FaceMesh + Attachable
│   ├── ModelObject        ← Box(RoundedBoxGeometry) 或 GLTF Mesh
│   ├── Light              ← SpotLight + 图标 Mesh
│   ├── Volume             ← 体积标记（半透明 Box）
│   ├── Image              ← PlaneGeometry + 纹理
│   ├── Ground             ← PlaneGeometry
│   ├── Room               ← BoxGeometry（可选）
│   └── Environment        ← GLTF 背景环境
├── 交互
│   ├── InteractionManager ← 射线拾取 + 多选 + 拖拽
│   ├── SGIkHelper         ← IK 端点可视化 + 拖拽
│   └── ObjectRotationControl ← 旋转 Gizmo
└── 效果
    ├── OutlineEffect      ← 描边（基于 postprocessing 的 Three.js 扩展）
    └── TWEEN              ← 相机动画补间

SceneManagerR3fSmall（俯视图）
├── ModelObject（仅 icon）
├── CameraIcon
├── IconsComponent         ← 角色/灯光/体积/图像 的俯视图标
└── Room（可选）
```

### 1.5 与 Electron 的耦合点（原始）

```javascript
// IPC 发送（renderer → main）
ipcRenderer.send('shot-generator:window:loaded')
ipcRenderer.send('saveShot', { uid, data, images })
ipcRenderer.send('insertShot', { ... })

// IPC 接收（main → renderer）
ipcRenderer.on('shot-generator:reload', (e, boardData) => { ... })
ipcRenderer.on('shot-generator:updateStore', (e, action) => { ... })
ipcRenderer.on('shot-generator:edit:undo', () => store.dispatch(ActionCreators.undo()))
ipcRenderer.on('shot-generator:edit:redo', () => store.dispatch(ActionCreators.redo()))

// 文件系统
const remote = require('@electron/remote')
remote.app.getPath('userData')           // 用户预设路径
path.join(window.__dirname, 'data', ...) // 内置资源路径
fs.readFileSync / fs.writeFileSync       // 设置持久化
```

---

## 2. 目标架构设计

### 2.1 整体布局

```
FlowCanvas 主窗口 (Electron BrowserWindow)
│
├── Konva 无限画布
│   └── ShotGeneratorNode (Konva.Group)   ← 新节点类型
│       ├── 背景矩形
│       ├── 3D 场景缩略图 (Konva.Image)
│       ├── 节点标题
│       └── 连接端口（与其他节点连线）
│
└── ShotGeneratorEditor (React Portal, z-index: 9999)
    ├── 仅当 isEditorOpen === true 时渲染
    ├── 双 Canvas 布局
    │   ├── 俯视图 Canvas (正交投影, 高度固定 180px)
    │   └── 主相机视图 Canvas (透视投影, 填满剩余空间)
    ├── Toolbar (顶部, 44px)
    ├── ElementsPanel (左侧, 220px)
    └── InspectorPanel (右侧, 280px)
```

### 2.2 状态管理策略

| 状态类型 | 存储位置 | 说明 |
|---------|---------|------|
| 场景对象 / 世界属性 | `useShotGeneratorStore`（Zustand + immer） | 每个节点独立存一份 |
| 撤销/重做历史 | 同上，手动维护 past/future 栈（最多 50 步） | 按节点独立 |
| 选择状态 / 骨骼选择 | `useShotGeneratorStore`（UI 状态） | 全局共享 |
| 预设数据 | `useShotGeneratorStore`（presets 字段） | 全局共享，从磁盘加载一次 |
| 节点位置 / 连线 | FlowCanvas 现有 Zustand store | 不改动 |
| 缩略图 | 节点数据（FlowCanvas DB） | 保存时由 canvas.toDataURL() 生成 |
| 场景持久化 | SQLite via FastAPI | 节点 ID 为键 |

### 2.3 数据流

```
用户双击节点
    │
    ├─ useShotGeneratorStore.openEditor(nodeId)
    ├─ 若 scenes[nodeId] 不存在 → 从 FastAPI 加载 or 创建默认场景
    └─ ShotGeneratorEditor 渲染

用户操作 3D 场景（拖拽角色、调整灯光等）
    │
    ├─ InteractionManager 计算新位置
    ├─ useShotGeneratorStore.updateObject(nodeId, objectId, newProps)
    └─ Zustand 推入历史栈 → R3F 组件 re-render

用户按 Cmd+Z
    │
    └─ useShotGeneratorStore.undo() → 弹出 past 栈顶到 present

用户点击 Save
    │
    ├─ canvas.toDataURL('image/jpeg', 0.85) → thumbnail
    ├─ useSaveShotScene(nodeId).mutate({ scene_data, thumbnail })
    │   └─ PUT /api/nodes/:nodeId/shot-scene
    ├─ 更新 FlowCanvas 节点的 thumbnail 字段
    └─ closeEditor()
```

---

## 3. 文件结构

```
src/
├── stores/
│   └── useShotGeneratorStore.ts          ← Zustand store（场景状态 + undo/redo）
│
├── nodes/
│   └── ShotGeneratorNode/
│       ├── index.tsx                     ← Konva 节点组件（缩略图展示）
│       ├── ShotGeneratorEditor.tsx       ← 全屏编辑器 Overlay（React Portal）
│       ├── SceneManagerLarge.tsx         ← 主相机视图 R3F 场景（从 R3F v4 迁移）
│       ├── SceneManagerSmall.tsx         ← 俯视图 R3F 场景（从 R3F v4 迁移）
│       ├── CameraUpdate.tsx              ← 同步相机属性
│       │
│       ├── components/
│       │   ├── Toolbar.tsx
│       │   ├── ElementsPanel.tsx
│       │   ├── InspectorPanel.tsx
│       │   │
│       │   ├── Three/                    ← Three.js 场景对象组件
│       │   │   ├── Character.tsx         ← SkinnedMesh + 骨骼 + IK（最复杂）
│       │   │   ├── ModelObject.tsx       ← Box / GLTF 普通道具
│       │   │   ├── Light.tsx             ← SpotLight + 图标
│       │   │   ├── Volume.tsx
│       │   │   ├── Image.tsx
│       │   │   ├── Ground.tsx
│       │   │   ├── Room.tsx
│       │   │   ├── Environment.tsx
│       │   │   ├── Attachable.tsx
│       │   │   ├── Group.tsx
│       │   │   ├── InteractionManager.tsx
│       │   │   ├── CameraControlsComponent.tsx
│       │   │   └── Icons/
│       │   │       ├── CameraIcon.tsx
│       │   │       └── IconsComponent.tsx
│       │   │
│       │   └── Inspector/
│       │       ├── GeneralInspector.tsx
│       │       ├── ModelInspector.tsx
│       │       ├── BoneInspector.tsx
│       │       ├── HandInspector.tsx
│       │       ├── CharacterPresetEditor.tsx
│       │       ├── CameraPanelInspector.tsx
│       │       └── GuidesInspector.tsx
│       │
│       ├── hooks/
│       │   ├── useAssetsManager.ts       ← GLTF/Texture 缓存（从 use-assets-manager.js 迁移）
│       │   ├── useDraggingManager.ts
│       │   ├── useShadingEffect.ts
│       │   ├── useSaveToFlowCanvas.ts    ← 替代 use-save-to-storyboarder.js
│       │   └── useUIScale.ts
│       │
│       ├── helpers/
│       │   ├── outlineMaterial.ts        ← MeshToonMaterial + 描边参数
│       │   ├── cloneGltf.ts              ← 深度克隆 GLTF（含骨骼绑定）
│       │   ├── traverseMeshMaterials.ts
│       │   └── isUserModel.ts
│       │
│       ├── utils/
│       │   ├── cameraUtils.ts            ← ShotSize / ShotAngle 预设
│       │   ├── gltfLoader.ts             ← GLTFLoader 单例
│       │   ├── ShotLayers.ts             ← Three.js Layer 常量
│       │   └── ThumbnailRenderer.ts      ← 离屏渲染缩略图
│       │
│       ├── contexts/
│       │   └── FilepathsContext.ts       ← 资源路径解析 Context
│       │
│       └── services/
│           └── filepaths.ts              ← 路径解析工厂函数
│
├── api/
│   └── shotGenerator.ts                  ← React Query hooks（save/load）
│
└── types/
    └── shotGenerator.ts                  ← TypeScript 类型定义

server/
└── routers/
    └── shot_generator.py                 ← FastAPI 路由
```

---

## 4. 核心实现：Zustand Store

```typescript
// src/stores/useShotGeneratorStore.ts
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'
import { nanoid } from 'nanoid'

// ─── 类型定义 ────────────────────────────────────────────────

export interface Vec3 { x: number; y: number; z: number }

export type SceneObjectType =
  | 'character' | 'object' | 'light' | 'camera'
  | 'volume' | 'image' | 'group' | 'attachable'

export interface BaseSceneObject {
  id: string
  type: SceneObjectType
  x: number; y: number; z: number
  rotation: number
  visible: boolean
  locked: boolean
  name?: string
}

export interface CharacterObject extends BaseSceneObject {
  type: 'character'
  model: string
  height: number
  headScale: number
  tintColor: string
  morphTargets: { mesomorphic: number; ectomorphic: number; endomorphic: number }
  skeleton: Record<string, { rotation: Vec3 }>
  handSkeleton: Record<string, { rotation: Vec3 }>
  poleTargets: Record<string, { position: Vec3; rotation: Vec3 }>
  ragDoll: boolean
  characterPresetId?: string
  posePresetId?: string
  handPosePresetId?: string
  emotionPresetId?: string
}

export interface ModelObject extends BaseSceneObject {
  type: 'object'
  model: string
  width: number; height: number; depth: number
  tintColor: string
}

export interface LightObject extends BaseSceneObject {
  type: 'light'
  intensity: number
  angle: number
  distance: number
  penumbra: number
  decay: number
  tilt: number
  roll: number
}

export interface CameraObject extends BaseSceneObject {
  type: 'camera'
  fov: number
  roll: number
}

export type SceneObject = CharacterObject | ModelObject | LightObject | CameraObject | BaseSceneObject

export interface WorldState {
  ground: boolean
  backgroundColor: string
  room: { visible: boolean; width: number; length: number; height: number }
  environment: { file: string; x: number; y: number; z: number; rotation: number; scale: number; visible: boolean }
  ambient: { intensity: number }
  directional: { intensity: number; rotation: number; tilt: number }
  fog: { visible: boolean; far: number }
  shadingMode: 'outline' | 'solid' | 'flat' | 'grayscale'
}

export interface ShotScene {
  sceneObjects: Record<string, SceneObject>
  world: WorldState
  activeCamera: string
}

interface ShotSceneHistory {
  present: ShotScene
  past: ShotScene[]
  future: ShotScene[]
}

// ─── 默认值 ──────────────────────────────────────────────────

const DEFAULT_WORLD: WorldState = {
  ground: true,
  backgroundColor: '#171717',
  room: { visible: false, width: 10, length: 10, height: 4 },
  environment: { file: '', x: 0, y: 0, z: 0, rotation: 0, scale: 1, visible: false },
  ambient: { intensity: 0.5 },
  directional: { intensity: 0.5, rotation: 0, tilt: 0 },
  fog: { visible: false, far: 40 },
  shadingMode: 'outline',
}

export const createDefaultScene = (overrides?: Partial<ShotScene>): ShotScene => {
  const cameraId = nanoid()
  return {
    activeCamera: cameraId,
    world: DEFAULT_WORLD,
    sceneObjects: {
      [cameraId]: {
        id: cameraId, type: 'camera',
        x: 0, y: 0, z: 5,
        rotation: 0, visible: true, locked: false,
        fov: 22.8, roll: 0, name: 'Camera',
      } as CameraObject,
    },
    ...overrides,
  }
}

// ─── 历史栈辅助 ───────────────────────────────────────────────

const MAX_HISTORY = 50

const withHistory = (h: ShotSceneHistory, next: ShotScene): ShotSceneHistory => ({
  past: [...h.past.slice(-(MAX_HISTORY - 1)), h.present],
  present: next,
  future: [],
})

// ─── Store 接口 ───────────────────────────────────────────────

interface ShotGeneratorStore {
  scenes: Record<string, ShotSceneHistory>
  activeNodeId: string | null
  isEditorOpen: boolean
  selections: string[]
  selectedBone: { characterId: string; boneName: string } | null
  mainViewCamera: 'live' | 'ortho'

  // Editor 控制
  openEditor: (nodeId: string) => void
  closeEditor: () => void

  // 场景生命周期
  initScene: (nodeId: string, initial?: ShotScene) => void
  removeScene: (nodeId: string) => void
  loadScene: (nodeId: string, scene: ShotScene) => void

  // 对象操作（每次操作自动推历史）
  createObject: (nodeId: string, props: Omit<SceneObject, 'id'>) => string
  updateObject: (nodeId: string, objectId: string, props: Partial<SceneObject>) => void
  updateObjects: (nodeId: string, updates: Record<string, Partial<SceneObject>>) => void
  deleteObjects: (nodeId: string, ids: string[]) => void

  // 世界属性
  updateWorld: (nodeId: string, props: Partial<WorldState>) => void
  setActiveCamera: (nodeId: string, cameraId: string) => void

  // 选择
  setSelections: (ids: string[]) => void
  selectObject: (id: string | null) => void
  toggleSelection: (id: string) => void
  setSelectedBone: (data: { characterId: string; boneName: string } | null) => void

  // 视图
  setMainViewCamera: (mode: 'live' | 'ortho') => void

  // Undo / Redo（作用于当前激活节点）
  undo: () => void
  redo: () => void
  canUndo: () => boolean
  canRedo: () => boolean
}

// ─── Store 实现 ───────────────────────────────────────────────

export const useShotGeneratorStore = create<ShotGeneratorStore>()(
  immer((set, get) => ({
    scenes: {},
    activeNodeId: null,
    isEditorOpen: false,
    selections: [],
    selectedBone: null,
    mainViewCamera: 'live',

    openEditor: (nodeId) => set(s => {
      s.activeNodeId = nodeId
      s.isEditorOpen = true
      s.selections = []
      s.selectedBone = null
    }),

    closeEditor: () => set(s => { s.isEditorOpen = false }),

    initScene: (nodeId, initial) => set(s => {
      if (s.scenes[nodeId]) return
      s.scenes[nodeId] = { present: initial ?? createDefaultScene(), past: [], future: [] }
    }),

    removeScene: (nodeId) => set(s => { delete s.scenes[nodeId] }),

    loadScene: (nodeId, scene) => set(s => {
      s.scenes[nodeId] = { present: scene, past: [], future: [] }
    }),

    createObject: (nodeId, props) => {
      const id = nanoid()
      set(s => {
        const h = s.scenes[nodeId]
        if (!h) return
        const next: ShotScene = {
          ...h.present,
          sceneObjects: { ...h.present.sceneObjects, [id]: { ...props, id } as SceneObject },
        }
        s.scenes[nodeId] = withHistory(h, next)
      })
      return id
    },

    updateObject: (nodeId, objectId, props) => set(s => {
      const h = s.scenes[nodeId]
      if (!h?.present.sceneObjects[objectId]) return
      const next: ShotScene = {
        ...h.present,
        sceneObjects: {
          ...h.present.sceneObjects,
          [objectId]: { ...h.present.sceneObjects[objectId], ...props },
        },
      }
      s.scenes[nodeId] = withHistory(h, next)
    }),

    updateObjects: (nodeId, updates) => set(s => {
      const h = s.scenes[nodeId]
      if (!h) return
      const nextObjects = { ...h.present.sceneObjects }
      for (const [id, props] of Object.entries(updates)) {
        if (nextObjects[id]) nextObjects[id] = { ...nextObjects[id], ...props }
      }
      s.scenes[nodeId] = withHistory(h, { ...h.present, sceneObjects: nextObjects })
    }),

    deleteObjects: (nodeId, ids) => set(s => {
      const h = s.scenes[nodeId]
      if (!h) return
      const nextObjects = { ...h.present.sceneObjects }
      ids.forEach(id => delete nextObjects[id])
      s.scenes[nodeId] = withHistory(h, { ...h.present, sceneObjects: nextObjects })
    }),

    updateWorld: (nodeId, props) => set(s => {
      const h = s.scenes[nodeId]
      if (!h) return
      s.scenes[nodeId] = withHistory(h, { ...h.present, world: { ...h.present.world, ...props } })
    }),

    setActiveCamera: (nodeId, cameraId) => set(s => {
      const h = s.scenes[nodeId]
      if (!h) return
      s.scenes[nodeId] = withHistory(h, { ...h.present, activeCamera: cameraId })
    }),

    setSelections: (ids) => set(s => { s.selections = ids }),

    selectObject: (id) => set(s => { s.selections = id ? [id] : [] }),

    toggleSelection: (id) => set(s => {
      const idx = s.selections.indexOf(id)
      if (idx === -1) s.selections.push(id)
      else s.selections.splice(idx, 1)
    }),

    setSelectedBone: (data) => set(s => { s.selectedBone = data }),

    setMainViewCamera: (mode) => set(s => { s.mainViewCamera = mode }),

    undo: () => set(s => {
      const nodeId = s.activeNodeId
      if (!nodeId) return
      const h = s.scenes[nodeId]
      if (!h?.past.length) return
      s.scenes[nodeId] = {
        past: h.past.slice(0, -1),
        present: h.past[h.past.length - 1],
        future: [h.present, ...h.future.slice(0, MAX_HISTORY - 1)],
      }
    }),

    redo: () => set(s => {
      const nodeId = s.activeNodeId
      if (!nodeId) return
      const h = s.scenes[nodeId]
      if (!h?.future.length) return
      s.scenes[nodeId] = {
        past: [...h.past.slice(-(MAX_HISTORY - 1)), h.present],
        present: h.future[0],
        future: h.future.slice(1),
      }
    }),

    canUndo: () => {
      const nodeId = get().activeNodeId
      if (!nodeId) return false
      return (get().scenes[nodeId]?.past.length ?? 0) > 0
    },

    canRedo: () => {
      const nodeId = get().activeNodeId
      if (!nodeId) return false
      return (get().scenes[nodeId]?.future.length ?? 0) > 0
    },
  }))
)

// ─── 便捷 Selector Hooks ──────────────────────────────────────

export const useActiveScene = (nodeId: string) =>
  useShotGeneratorStore(s => s.scenes[nodeId]?.present)

export const useSceneObjects = (nodeId: string) =>
  useShotGeneratorStore(s => s.scenes[nodeId]?.present.sceneObjects ?? {})

export const useWorld = (nodeId: string) =>
  useShotGeneratorStore(s => s.scenes[nodeId]?.present.world ?? DEFAULT_WORLD)

export const useActiveCamera = (nodeId: string) =>
  useShotGeneratorStore(s => s.scenes[nodeId]?.present.activeCamera ?? '')
```

---

## 5. 核心实现：Konva 节点

```typescript
// src/nodes/ShotGeneratorNode/index.tsx
import React, { useEffect } from 'react'
import { Group, Rect, Image, Text, Circle } from 'react-konva'
import useImage from 'use-image'
import { useShotGeneratorStore } from '../../stores/useShotGeneratorStore'

const NODE_WIDTH = 320
const NODE_HEIGHT = 200
const HEADER_HEIGHT = 24

interface ShotGeneratorNodeProps {
  id: string
  x: number
  y: number
  thumbnail?: string    // base64 JPEG，保存时由 Editor 更新
  onDragEnd?: (x: number, y: number) => void
}

export const ShotGeneratorNode: React.FC<ShotGeneratorNodeProps> = ({
  id, x, y, thumbnail, onDragEnd
}) => {
  const { initScene, openEditor } = useShotGeneratorStore()
  const [thumbImage] = useImage(thumbnail ?? '')

  // 懒初始化：节点首次出现时创建空场景
  useEffect(() => { initScene(id) }, [id])

  return (
    <Group
      x={x} y={y}
      draggable
      onDragEnd={e => onDragEnd?.(e.target.x(), e.target.y())}
      onDblClick={() => openEditor(id)}
    >
      {/* 节点背景 */}
      <Rect
        width={NODE_WIDTH} height={NODE_HEIGHT}
        fill="#1a1a2e" stroke="#4a4a8a" strokeWidth={1.5}
        cornerRadius={8} shadowColor="black"
        shadowBlur={12} shadowOpacity={0.4}
      />

      {/* 标题栏 */}
      <Rect
        width={NODE_WIDTH} height={HEADER_HEIGHT}
        fill="#2a2a5a" cornerRadius={[8, 8, 0, 0]}
      />
      <Text
        text="☰  Shot Generator"
        x={10} y={6}
        fontSize={11} fill="#a0a0e0" fontStyle="500"
      />

      {/* 3D 场景缩略图 */}
      {thumbImage ? (
        <Image
          image={thumbImage}
          x={4} y={HEADER_HEIGHT + 2}
          width={NODE_WIDTH - 8} height={NODE_HEIGHT - HEADER_HEIGHT - 6}
          cornerRadius={4}
        />
      ) : (
        <Rect
          x={4} y={HEADER_HEIGHT + 2}
          width={NODE_WIDTH - 8} height={NODE_HEIGHT - HEADER_HEIGHT - 6}
          fill="#0d0d20" cornerRadius={4}
        />
      )}

      {/* 双击提示（无缩略图时） */}
      {!thumbnail && (
        <Text
          text="Double-click to edit"
          x={NODE_WIDTH / 2} y={NODE_HEIGHT / 2 + 8}
          align="center" offsetX={NODE_WIDTH / 2}
          fontSize={11} fill="#505080"
        />
      )}

      {/* 输出端口（右侧中心） */}
      <Circle
        x={NODE_WIDTH} y={NODE_HEIGHT / 2}
        radius={5} fill="#4a4aaa" stroke="#8080ff" strokeWidth={1.5}
      />
    </Group>
  )
}
```

---

## 6. 核心实现：编辑器 Overlay

```typescript
// src/nodes/ShotGeneratorNode/ShotGeneratorEditor.tsx
import React, { useCallback, useEffect, useRef } from 'react'
import { createPortal } from 'react-dom'
import { Canvas } from '@react-three/fiber'
import { useShotGeneratorStore } from '../../stores/useShotGeneratorStore'
import { SceneManagerLarge } from './SceneManagerLarge'
import { SceneManagerSmall } from './SceneManagerSmall'
import { Toolbar } from './components/Toolbar'
import { ElementsPanel } from './components/ElementsPanel'
import { InspectorPanel } from './components/InspectorPanel'
import { useSaveShotScene } from '../../api/shotGenerator'

export const ShotGeneratorEditor: React.FC = () => {
  const { isEditorOpen, activeNodeId, closeEditor, undo, redo, canUndo, canRedo } =
    useShotGeneratorStore()
  const mainGLRef = useRef<WebGLRenderingContext | null>(null)
  const saveMutation = useSaveShotScene(activeNodeId ?? '')

  // 键盘快捷键
  useEffect(() => {
    if (!isEditorOpen) return
    const onKey = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'z') {
        e.shiftKey ? redo() : undo()
      }
      if (e.key === 'Escape') closeEditor()
    }
    window.addEventListener('keydown', onKey)
    return () => window.removeEventListener('keydown', onKey)
  }, [isEditorOpen, undo, redo, closeEditor])

  const handleSave = useCallback(async () => {
    if (!mainGLRef.current || !activeNodeId) return
    const canvas = (mainGLRef.current as any).domElement as HTMLCanvasElement
    const thumbnail = canvas.toDataURL('image/jpeg', 0.85)
    // 从 store 获取当前场景数据
    const scene = useShotGeneratorStore.getState().scenes[activeNodeId]?.present
    if (!scene) return
    await saveMutation.mutateAsync({ scene_data: scene, thumbnail })
    closeEditor()
  }, [activeNodeId, saveMutation, closeEditor])

  if (!isEditorOpen || !activeNodeId) return null

  return createPortal(
    <div style={{
      position: 'fixed', inset: 0, zIndex: 9999,
      display: 'grid',
      gridTemplate: `
        "toolbar  toolbar   toolbar"  44px
        "elements main      inspector" 1fr
        / 220px   1fr       280px
      `,
      background: '#111',
      fontFamily: 'system-ui, sans-serif',
    }}>
      {/* 工具栏 */}
      <div style={{ gridArea: 'toolbar', borderBottom: '1px solid #2a2a4a' }}>
        <Toolbar
          nodeId={activeNodeId}
          onSave={handleSave}
          onClose={closeEditor}
          canUndo={canUndo()}
          canRedo={canRedo()}
          onUndo={undo}
          onRedo={redo}
          isSaving={saveMutation.isPending}
        />
      </div>

      {/* 对象列表 */}
      <div style={{ gridArea: 'elements', borderRight: '1px solid #2a2a4a', overflow: 'auto' }}>
        <ElementsPanel nodeId={activeNodeId} />
      </div>

      {/* 3D 视图 */}
      <div style={{ gridArea: 'main', display: 'flex', flexDirection: 'column' }}>
        {/* 俯视图（固定高度） */}
        <div style={{ height: 180, borderBottom: '1px solid #2a2a4a', flexShrink: 0 }}>
          <Canvas
            orthographic
            camera={{ zoom: 60, position: [0, 16, 0], up: [0, 0, -1], near: 0.01, far: 1000 }}
            dpr={window.devicePixelRatio}
          >
            <SceneManagerSmall nodeId={activeNodeId} />
          </Canvas>
        </div>

        {/* 主相机视图 */}
        <div style={{ flex: 1, position: 'relative' }}>
          <Canvas
            camera={{ fov: 22.8, near: 0.01, far: 1000, position: [0, 0, 5] }}
            dpr={window.devicePixelRatio}
            gl={{ preserveDrawingBuffer: true, antialias: true }}
            onCreated={({ gl }) => { mainGLRef.current = gl as any }}
          >
            <SceneManagerLarge nodeId={activeNodeId} />
          </Canvas>
        </div>
      </div>

      {/* 属性检查器 */}
      <div style={{ gridArea: 'inspector', borderLeft: '1px solid #2a2a4a', overflow: 'auto' }}>
        <InspectorPanel nodeId={activeNodeId} />
      </div>
    </div>,
    document.body
  )
}
```

---

## 7. 核心实现：R3F 场景管理器

### SceneManagerLarge.tsx（主相机视图）

原始文件：`src/js/shot-generator/SceneManagerR3fLarge.js`（~600行）

移植要点：
- `connect()(Component)` → 直接用 Zustand selector hooks
- `useUpdate(cb, deps)` → `useEffect` + `ref`
- `import { Canvas } from 'react-three-fiber'` → `import { Canvas } from '@react-three/fiber'`

```typescript
// src/nodes/ShotGeneratorNode/SceneManagerLarge.tsx
import React, { useRef, useEffect, useMemo, useCallback } from 'react'
import { useThree, useFrame } from '@react-three/fiber'   // ← v8 新包名
import * as THREE from 'three'
import TWEEN from '@tweenjs/tween.js'
import { useSceneObjects, useWorld, useShotGeneratorStore } from '../../stores/useShotGeneratorStore'
import Character from './components/Three/Character'
import ModelObject from './components/Three/ModelObject'
import Light from './components/Three/Light'
import Volume from './components/Three/Volume'
import Image from './components/Three/Image'
import Ground from './components/Three/Ground'
import Room from './components/Three/Room'
import Environment from './components/Three/Environment'
import Attachable from './components/Three/Attachable'
import InteractionManager from './components/Three/InteractionManager'
import { CameraUpdate } from './CameraUpdate'
import { useShadingEffect } from './hooks/useShadingEffect'
import { FilepathsContext } from './contexts/FilepathsContext'

interface Props {
  nodeId: string
}

export const SceneManagerLarge: React.FC<Props> = ({ nodeId }) => {
  const sceneObjects = useSceneObjects(nodeId)
  const world = useWorld(nodeId)
  const { selections, selectObject } = useShotGeneratorStore()
  const rootRef = useRef<THREE.Group>(null)
  const { scene } = useThree()

  // Tween 动画循环
  useFrame(() => { TWEEN.update() })

  // 着色效果（描边 / 灰度 / 实心）
  useShadingEffect({ world, scene })

  // 分类筛选
  const characterIds = useMemo(
    () => Object.values(sceneObjects).filter(o => o.type === 'character').map(o => o.id),
    [sceneObjects]
  )
  const modelObjectIds = useMemo(
    () => Object.values(sceneObjects).filter(o => o.type === 'object').map(o => o.id),
    [sceneObjects]
  )
  const lightIds = useMemo(
    () => Object.values(sceneObjects).filter(o => o.type === 'light').map(o => o.id),
    [sceneObjects]
  )

  return (
    <group ref={rootRef}>
      {/* 环境光 */}
      <ambientLight intensity={world.ambient.intensity} />

      {/* 方向光 */}
      <directionalLight
        intensity={world.directional.intensity}
        rotation={[0, world.directional.rotation, world.directional.tilt]}
      />

      {/* 相机同步 */}
      <CameraUpdate nodeId={nodeId} />

      {/* 场景对象 */}
      {characterIds.map(id => (
        <Character
          key={id}
          nodeId={nodeId}
          sceneObject={sceneObjects[id] as any}
          isSelected={selections.includes(id)}
        />
      ))}
      {modelObjectIds.map(id => (
        <ModelObject
          key={id}
          nodeId={nodeId}
          sceneObject={sceneObjects[id] as any}
          isSelected={selections.includes(id)}
        />
      ))}
      {lightIds.map(id => (
        <Light
          key={id}
          nodeId={nodeId}
          sceneObject={sceneObjects[id] as any}
          isSelected={selections.includes(id)}
        />
      ))}

      {/* 地面 / 房间 / 环境 */}
      {world.ground && <Ground />}
      {world.room.visible && <Room room={world.room} />}
      {world.environment.visible && <Environment world={world} />}

      {/* 交互管理器（射线拾取、拖拽） */}
      <InteractionManager nodeId={nodeId} />

      {/* 雾效 */}
      {world.fog.visible && <fog args={['#000000', 0, world.fog.far]} />}
    </group>
  )
}
```

---

## 8. 核心实现：Three.js 组件移植

### ModelObject.tsx

原始文件：`src/js/shot-generator/components/Three/ModelObject.js`

关键变化：`useUpdate` → `useEffect + ref`，`extend` 用法不变

```typescript
// src/nodes/ShotGeneratorNode/components/Three/ModelObject.tsx
import React, { useRef, useMemo, useEffect } from 'react'
import * as THREE from 'three'
import { extend } from '@react-three/fiber'
import { useAsset } from '../../hooks/useAssetsManager'
import { SHOT_LAYERS } from '../../utils/ShotLayers'
import { patchMaterial, setSelected } from '../../helpers/outlineMaterial'
import RoundedBoxGeometryCreator from '../../../../vendor/three-rounded-box'
import { ModelObject as ModelObjectType } from '../../../../stores/useShotGeneratorStore'

const RoundedBoxGeometry = RoundedBoxGeometryCreator(THREE)
extend({ RoundedBoxGeometry })

const materialFactory = () => patchMaterial(new THREE.MeshToonMaterial({
  color: 0xcccccc,
  emissive: 0x0,
  shininess: 0,
  flatShading: false,
}))

interface Props {
  sceneObject: ModelObjectType
  isSelected: boolean
}

export const ModelObject: React.FC<Props> = ({ sceneObject, isSelected }) => {
  const ref = useRef<THREE.Group>(null)
  const { asset } = useAsset(sceneObject.model !== 'box' ? sceneObject.model : null)

  // ← v8: 替代 useUpdate
  useEffect(() => {
    ref.current?.traverse(child => child.layers.enable(SHOT_LAYERS))
  })

  useEffect(() => {
    ref.current?.traverse(child => {
      if ((child as THREE.Mesh).isMesh) {
        setSelected(child as THREE.Mesh, isSelected)
      }
    })
  }, [isSelected])

  const { x, y, z, rotation, width = 1, height = 1, depth = 1 } = sceneObject

  return (
    <group
      ref={ref}
      position={[x, y + height / 2, z]}
      rotation={[0, rotation, 0]}
      scale={[width, height, depth]}
      userData={{ type: 'object', id: sceneObject.id }}
    >
      {sceneObject.model === 'box' ? (
        <mesh>
          {/* @ts-ignore — extend 注入的自定义几何体 */}
          <roundedBoxGeometry args={[1, 1, 1, 0.005, 5]} />
          <primitive object={materialFactory()} attach="material" />
        </mesh>
      ) : asset ? (
        asset.scene.children
          .filter((c: THREE.Object3D) => (c as THREE.Mesh).isMesh)
          .map((child: THREE.Object3D) => (
            <primitive key={child.uuid} object={child.clone()} />
          ))
      ) : null}
    </group>
  )
}
```

### Character.tsx（关键设计说明）

原始文件：`src/js/shot-generator/components/Three/Character.js`（约 350 行，最复杂的组件）

移植时需要特别注意：

1. **`useUpdate` 替换**：原始代码大量使用 `useUpdate` 来在挂载时对 LOD 对象设置 Layer，迁移后改为 `useEffect(() => { ... })` 无依赖版本（每帧后执行一次，等效于 componentDidMount/Update）。

2. **骨骼姿势应用**：在 `useEffect` 中遍历 `skeleton` 数据应用到 Three.js 骨骼。

3. **IK 系统**：当 `sceneObject.ragDoll === true` 时激活 SGIkHelper，否则走直接骨骼赋值。

4. **LOD 系统**：原始代码创建 `THREE.LOD` 对象并根据相机距离显示不同精度的网格。

5. **FaceMesh**：面部表情通过单独的 FaceMesh 实例驱动 morph target。

```typescript
// 核心结构概要（完整实现约 300 行）
export const Character: React.FC<CharacterProps> = ({ nodeId, sceneObject, isSelected }) => {
  const ref = useRef<THREE.Group>(null)
  const { asset: gltf } = useAsset(resolveCharacterPath(sceneObject.model))
  const { updateObject, selectedBone, setSelectedBone } = useShotGeneratorStore()

  // 1. 克隆 GLTF 并构建 LOD
  const [skeleton, lod] = useMemo(() => {
    if (!gltf) return [null, null]
    const { scene } = cloneGltf(gltf)
    const meshes = scene.children.filter((c: any) => c.isSkinnedMesh)
    // ... LOD 构建逻辑（同原始代码）
    return [skeleton, lod]
  }, [gltf])

  // 2. 应用骨骼姿势
  useEffect(() => {
    if (!skeleton || !sceneObject.skeleton) return
    for (const [boneName, { rotation }] of Object.entries(sceneObject.skeleton)) {
      const bone = skeleton.bones.find((b: THREE.Bone) => b.name === boneName)
      if (bone) bone.rotation.set(rotation.x, rotation.y, rotation.z)
    }
  }, [skeleton, sceneObject.skeleton])

  // 3. IK Helper（当 ragDoll 开启时）
  useEffect(() => {
    if (!sceneObject.ragDoll || !ref.current || !skeleton) return
    const ikHelper = SGIkHelper.getInstance()
    ref.current.add(ikHelper)
    ikHelper.initialize(ref.current, sceneObject.height, skeleton.skinnedMesh, sceneObject)
    return () => { ref.current?.remove(ikHelper) }
  }, [sceneObject.ragDoll, skeleton])

  // 4. 选中效果
  useEffect(() => {
    ref.current?.traverse(child => {
      if ((child as THREE.Mesh).isMesh) setSelected(child as THREE.Mesh, isSelected)
    })
  }, [isSelected])

  return (
    <group ref={ref} position={[sceneObject.x, 0, sceneObject.z]} rotation={[0, sceneObject.rotation, 0]}>
      {lod && <primitive object={lod} />}
    </group>
  )
}
```

---

## 9. 核心实现：IK 系统适配

### 问题：Singleton 模式的限制

原始 `SGIkHelper` 是全局单例（`new SGIkHelper()` 返回同一个实例），这在单窗口单场景的情况下可行。移植到 FlowCanvas 后，同一时刻可能有多个节点的场景处于活跃状态，单例会冲突。

### 解决方案：per-scene 实例化

```typescript
// src/nodes/ShotGeneratorNode/hooks/useIkHelper.ts
import { useRef, useEffect } from 'react'
import * as THREE from 'three'

// 不再使用 Singleton，改为 useRef 持有实例
export const useIkHelper = (nodeId: string) => {
  const ikHelperRef = useRef<any>(null)

  useEffect(() => {
    // 动态 import（保持与原始 SGIkHelper 代码的兼容）
    import('../../../../shared/IK/SGIkHelper').then(({ default: SGIkHelperClass }) => {
      // 创建独立实例（需要修改 SGIkHelper 移除 Singleton 模式）
      ikHelperRef.current = new SGIkHelperClass()
    })
    return () => {
      ikHelperRef.current = null
    }
  }, [nodeId])

  return ikHelperRef
}
```

**SGIkHelper 的改造**（`src/js/shared/IK/SGIkHelper.js` 中需要修改）：

```javascript
// 原始（Singleton 模式，不支持多实例）
let instance = null
class SGIKHelper extends THREE.Object3D {
  constructor() {
    if (!instance) { super(); instance = this }
    return instance
  }
  static getInstance() { return instance ? instance : new SGIKHelper() }
}

// 改造后（支持多实例，同时保持向后兼容）
class SGIKHelper extends THREE.Object3D {
  constructor() { super() }
  // 移除 static getInstance()，或保留仅用于 Storyboarder 原始代码兼容
}
```

---

## 10. 核心实现：资源加载系统

原始文件：`src/js/shot-generator/hooks/use-assets-manager.js`

核心逻辑：用一个 observable 的全局 cache 对象缓存 GLTF 和纹理，同一路径只加载一次。

```typescript
// src/nodes/ShotGeneratorNode/hooks/useAssetsManager.ts
import { useState, useEffect } from 'react'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'

type AssetStatus = 'pending' | 'success' | 'error'

interface CacheEntry {
  data: any
  status: AssetStatus
}

// 全局 cache（单例，跨所有节点共享，避免重复加载）
const cache = new Map<string, CacheEntry>()
const listeners = new Map<string, Set<() => void>>()

const notify = (path: string) => {
  listeners.get(path)?.forEach(fn => fn())
}

const gltfLoader = new GLTFLoader()
const textureLoader = new THREE.TextureLoader()

export const loadAsset = (path: string): Promise<any> => {
  if (!path) return Promise.resolve(null)

  const cached = cache.get(path)
  if (cached?.status === 'success') return Promise.resolve(cached.data)
  if (cached?.status === 'pending') {
    return new Promise(resolve => {
      const check = () => {
        const c = cache.get(path)
        if (c?.status === 'success') resolve(c.data)
      }
      const set = listeners.get(path) ?? new Set()
      set.add(check)
      listeners.set(path, set)
    })
  }

  cache.set(path, { data: null, status: 'pending' })

  const loader = path.match(/\.(glb|gltf)$/i) ? gltfLoader : textureLoader

  return new Promise((resolve, reject) => {
    loader.load(
      path,
      (data) => {
        cache.set(path, { data, status: 'success' })
        notify(path)
        resolve(data)
      },
      undefined,
      (err) => {
        cache.set(path, { data: null, status: 'error' })
        notify(path)
        reject(err)
      }
    )
  })
}

export const useAsset = (path: string | null) => {
  const [entry, setEntry] = useState<CacheEntry | null>(
    path ? (cache.get(path) ?? null) : null
  )

  useEffect(() => {
    if (!path) return
    const current = cache.get(path)
    if (current?.status === 'success') {
      setEntry(current)
      return
    }

    loadAsset(path).then(() => {
      setEntry(cache.get(path) ?? null)
    })

    const update = () => setEntry(cache.get(path) ?? null)
    const set = listeners.get(path) ?? new Set()
    set.add(update)
    listeners.set(path, set)

    return () => { listeners.get(path)?.delete(update) }
  }, [path])

  return { asset: entry?.status === 'success' ? entry.data : null }
}
```

### 资源路径解析

原始代码通过 `FilepathsContext` 提供路径解析函数，在 FlowCanvas 中需要适配路径来源：

```typescript
// src/nodes/ShotGeneratorNode/services/filepaths.ts
import path from 'path'
import { app } from '@electron/remote'

// 系统内置资源路径（随应用打包）
const DATA_DIR = path.join(app.getAppPath(), 'data', 'shot-generator')

// 用户项目资源路径（工作区目录下）
const getProjectDir = (workspaceDir: string) => path.join(workspaceDir, 'assets')

export const createAssetPathResolver = (workspaceDir: string) => {
  return (type: string, filepath: string = ''): string => {
    if (filepath.includes('/')) {
      // 用户自定义资源：从工作区目录查找
      return path.join(getProjectDir(workspaceDir), 'models', type, path.basename(filepath))
    } else {
      // 内置资源：从应用数据目录查找
      const typeDir: Record<string, string> = {
        character: path.join(DATA_DIR, 'dummies', 'gltf'),
        object: path.join(DATA_DIR, 'objects'),
        attachable: path.join(DATA_DIR, 'attachables'),
        emotion: path.join(DATA_DIR, 'emotions'),
        image: path.join(DATA_DIR, 'images'),
      }
      return path.join(typeDir[type] ?? DATA_DIR, filepath)
    }
  }
}
```

---

## 11. 核心实现：保存与持久化

原始文件：`src/js/shot-generator/hooks/use-save-to-storyboarder.js`

原始流程：取消选中 → 离屏渲染截图 → 序列化 state → `ipcRenderer.send('saveShot', ...)` → 主进程写文件

移植后流程：取消选中 → WebGL canvas.toDataURL() → Zustand 序列化 → React Query mutation → FastAPI 写 SQLite

```typescript
// src/nodes/ShotGeneratorNode/hooks/useSaveToFlowCanvas.ts
import { useCallback, useRef } from 'react'
import { useThree } from '@react-three/fiber'
import { useShotGeneratorStore } from '../../../stores/useShotGeneratorStore'
import { useSaveShotScene } from '../../../api/shotGenerator'

export const useSaveToFlowCanvas = (nodeId: string) => {
  const { gl } = useThree()
  const { selectObject, scenes } = useShotGeneratorStore()
  const saveMutation = useSaveShotScene(nodeId)

  const save = useCallback(async () => {
    // 1. 取消选中（隐藏 outline）
    selectObject(null)

    // 2. 等待一帧渲染（确保 outline 消失）
    await new Promise(resolve => requestAnimationFrame(resolve))

    // 3. 截图
    const thumbnail = (gl as any).domElement.toDataURL('image/jpeg', 0.85)

    // 4. 序列化场景（移除 runtime 字段）
    const scene = scenes[nodeId]?.present
    if (!scene) return

    const serialized = {
      world: scene.world,
      activeCamera: scene.activeCamera,
      sceneObjects: Object.fromEntries(
        Object.entries(scene.sceneObjects).map(([id, obj]) => {
          // 移除 runtime 字段（如 loaded、displayName）
          const { ...rest } = obj
          return [id, rest]
        })
      ),
    }

    // 5. 保存到 FastAPI
    await saveMutation.mutateAsync({ scene_data: serialized, thumbnail })
  }, [nodeId, gl, scenes, selectObject, saveMutation])

  return { save, isSaving: saveMutation.isPending }
}
```

---

## 12. 后端接口

### FastAPI 路由

```python
# server/routers/shot_generator.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from typing import Optional
import json, base64, sqlite3
from pathlib import Path

router = APIRouter(prefix="/api/nodes", tags=["shot-generator"])


class ShotScenePayload(BaseModel):
    scene_data: dict
    thumbnail: Optional[str] = None  # base64 JPEG data URL


def get_db():
    conn = sqlite3.connect("workspace/flowcanvas.db")
    conn.row_factory = sqlite3.Row
    return conn


# ─── 建表（应在 startup 中执行）────────────────────────────────
CREATE_TABLE = """
CREATE TABLE IF NOT EXISTS shot_scenes (
    node_id TEXT PRIMARY KEY,
    scene_data TEXT NOT NULL,
    thumbnail_path TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
)
"""


@router.get("/{node_id}/shot-scene")
async def get_shot_scene(node_id: str):
    db = get_db()
    row = db.execute(
        "SELECT scene_data FROM shot_scenes WHERE node_id = ?", [node_id]
    ).fetchone()
    db.close()
    if not row:
        raise HTTPException(404, detail="Scene not found")
    return json.loads(row["scene_data"])


@router.put("/{node_id}/shot-scene")
async def save_shot_scene(node_id: str, payload: ShotScenePayload):
    thumbnail_path = None

    # 保存缩略图到磁盘
    if payload.thumbnail:
        thumb_dir = Path(f"workspace/thumbnails")
        thumb_dir.mkdir(parents=True, exist_ok=True)
        # 解码 base64 data URL
        header, data = payload.thumbnail.split(",", 1)
        img_bytes = base64.b64decode(data)
        thumbnail_path = str(thumb_dir / f"{node_id}.jpg")
        Path(thumbnail_path).write_bytes(img_bytes)

    db = get_db()
    db.execute(
        """
        INSERT INTO shot_scenes (node_id, scene_data, thumbnail_path, updated_at)
        VALUES (?, ?, ?, CURRENT_TIMESTAMP)
        ON CONFLICT(node_id) DO UPDATE SET
            scene_data = excluded.scene_data,
            thumbnail_path = excluded.thumbnail_path,
            updated_at = CURRENT_TIMESTAMP
        """,
        [node_id, json.dumps(payload.scene_data), thumbnail_path],
    )
    db.commit()
    db.close()
    return {"status": "ok", "thumbnail_path": thumbnail_path}


@router.get("/{node_id}/thumbnail")
async def get_thumbnail(node_id: str):
    from fastapi.responses import FileResponse
    db = get_db()
    row = db.execute(
        "SELECT thumbnail_path FROM shot_scenes WHERE node_id = ?", [node_id]
    ).fetchone()
    db.close()
    if not row or not row["thumbnail_path"]:
        raise HTTPException(404)
    return FileResponse(row["thumbnail_path"], media_type="image/jpeg")
```

### React Query Hooks

```typescript
// src/api/shotGenerator.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { ShotScene } from '../stores/useShotGeneratorStore'

const BASE = '/api/nodes'

export const useShotScene = (nodeId: string) =>
  useQuery<ShotScene>({
    queryKey: ['shot-scene', nodeId],
    queryFn: () => fetch(`${BASE}/${nodeId}/shot-scene`).then(r => {
      if (!r.ok) throw new Error('Not found')
      return r.json()
    }),
    enabled: !!nodeId,
    staleTime: Infinity,  // 场景数据由 mutation 主动失效，不自动 refetch
  })

export const useSaveShotScene = (nodeId: string) => {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: { scene_data: ShotScene; thumbnail: string }) =>
      fetch(`${BASE}/${nodeId}/shot-scene`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      }).then(r => r.json()),
    onSuccess: () => {
      // 失效缓存，触发节点缩略图刷新
      qc.invalidateQueries({ queryKey: ['shot-scene', nodeId] })
      qc.invalidateQueries({ queryKey: ['node-thumbnail', nodeId] })
    },
  })
}
```

---

## 13. R3F v4 → v8 迁移手册

这是移植过程中技术风险最大的部分。

### 包名变更

```bash
# 卸载旧版
npm uninstall react-three-fiber

# 安装新版
npm install @react-three/fiber @react-three/drei three
npm install -D @types/three
```

### API 对照表

| v4 写法 | v8 写法 | 说明 |
|---------|---------|------|
| `import { Canvas, useThree, useFrame } from 'react-three-fiber'` | `import { Canvas, useThree, useFrame } from '@react-three/fiber'` | 包名变更 |
| `<Canvas pixelRatio={window.devicePixelRatio}>` | `<Canvas dpr={window.devicePixelRatio}>` | prop 重命名 |
| `<Canvas colorManagement>` | 删除（v8 默认开启） | — |
| `<Canvas concurrent>` | 删除（v8 默认开启） | — |
| `useUpdate(cb, deps)` | `useEffect(() => { ref.current && cb(ref.current) }, deps)` | Hook 已移除 |
| `useResource()` | `useRef<T>(null)` | Hook 已移除 |
| `ReactThreeFiber.Object3DNode` | `ThreeElements` | 类型重命名 |
| `state.gl.setSize(w, h)` | `state.gl.setSize(w, h)` | 不变 |
| `state.invalidate()` | `state.invalidate()` | 不变 |
| `extend({ MyGeometry })` | `extend({ MyGeometry })` | 不变 |

### useUpdate 替换模式

```typescript
// ─── 原始 v4 写法 ────────────────────────────────────────────
const ref = useUpdate<THREE.Mesh>(
  self => { self.layers.enable(SHOT_LAYERS) },
  []  // 空依赖：仅挂载时执行
)

// ─── v8 写法 ─────────────────────────────────────────────────
const ref = useRef<THREE.Mesh>(null)
useEffect(() => {
  if (ref.current) ref.current.layers.enable(SHOT_LAYERS)
}, [])  // 等效于挂载时执行一次

// 如果需要每次 prop 变化时执行（有依赖项）：
useEffect(() => {
  if (ref.current) ref.current.layers.enable(SHOT_LAYERS)
})  // 无依赖数组 = 每次 render 后执行（最接近原始 useUpdate 行为）
```

### Canvas 截图方式变更

```typescript
// v4 写法
const { gl } = useThree()
const dataUrl = gl.domElement.toDataURL()

// v8 写法（相同，但需要在 Canvas 上设置 preserveDrawingBuffer）
<Canvas gl={{ preserveDrawingBuffer: true }}>
// 在组件内：
const { gl } = useThree()
const dataUrl = gl.domElement.toDataURL('image/jpeg', 0.85)
```

### 已不再需要的依赖

```bash
# v8 已内置，不需要单独安装：
# - postprocessing（OutlineEffect 改为 @react-three/postprocessing）
# - three-stdlib（已内置）
```

---

## 14. 执行计划

### Phase 1：基础搭建（Day 1~2）

- [ ] 安装 `@react-three/fiber` v8、`@react-three/drei`、`three`、`@types/three`
- [ ] 创建 `src/stores/useShotGeneratorStore.ts`（完整实现，含 undo/redo）
- [ ] 创建 `src/nodes/ShotGeneratorNode/index.tsx`（Konva 节点，带缩略图）
- [ ] 创建 `ShotGeneratorEditor.tsx` 骨架（双 Canvas 布局，Portal 渲染）
- [ ] FastAPI `shot_generator.py` 路由 + SQLite 建表
- [ ] React Query hooks (`useShotScene`, `useSaveShotScene`)

### Phase 2：3D 场景（Day 3~5）

- [ ] `SceneManagerLarge.tsx`（主相机视图，R3F v8 适配）
- [ ] `SceneManagerSmall.tsx`（俯视图）
- [ ] `CameraUpdate.tsx`
- [ ] `ModelObject.tsx`（Box + GLTF）
- [ ] `Light.tsx`（SpotLight）
- [ ] `Ground.tsx`、`Room.tsx`、`Environment.tsx`
- [ ] `useAssetsManager.ts`（GLTF/Texture 缓存）
- [ ] `FilepathsContext.ts` + `filepaths.ts`（路径解析）

### Phase 3：角色与 IK（Day 6~7）

- [ ] `Character.tsx`（SkinnedMesh + LOD + 骨骼姿势应用）
- [ ] `Attachable.tsx`
- [ ] `cloneGltf.ts`（深度克隆含骨骼绑定）
- [ ] `outlineMaterial.ts`（MeshToonMaterial + 描边）
- [ ] SGIkHelper 改造（移除 Singleton → 支持多实例）
- [ ] `useIkHelper.ts`

### Phase 4：Inspector UI（Day 8~9）

- [ ] `Toolbar.tsx`（工具选择、保存、关闭、undo/redo 按钮）
- [ ] `ElementsPanel.tsx`（对象树列表）
- [ ] `InspectorPanel.tsx`（通用属性：位置、旋转、可见性）
- [ ] `GeneralInspector.tsx`
- [ ] `ModelInspector.tsx`
- [ ] `BoneInspector.tsx`（骨骼姿势编辑）
- [ ] `CameraPanelInspector.tsx`（FOV、ShotSize、ShotAngle）

### Phase 5：交互与集成（Day 10）

- [ ] `InteractionManager.tsx`（射线拾取、多选、拖拽，R3F v8 适配）
- [ ] 键盘快捷键（Cmd+Z/Y, Delete）
- [ ] 保存流程端到端测试（截图 → FastAPI → Konva 缩略图更新）
- [ ] 与 FlowCanvas 现有节点系统集成（注册新节点类型）

### 风险提示

| 风险 | 影响 | 应对 |
|------|------|------|
| R3F v4 → v8 `useUpdate` 大量替换 | 中 | 搜索 `useUpdate` 全量替换，优先处理渲染相关的 |
| SGIkHelper Singleton 多实例冲突 | 高 | Phase 3 第一件事，先解决再做 Character |
| OutlineEffect 在 R3F v8 中的集成 | 中 | 改用 `@react-three/postprocessing` 的 `Outline` |
| `window.__dirname` 在渲染进程中不可用（vite 打包） | 低 | 通过 `@electron/remote` 的 `app.getAppPath()` 替代 |
