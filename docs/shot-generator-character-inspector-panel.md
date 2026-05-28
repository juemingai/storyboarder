# Shot Generator 人物属性面板（Character Inspector Panel）功能分析

> 本文档面向 FlowCanvas 3D 导演台功能的移植工作，完整描述 Shot Generator 中选中人物时左侧属性面板的所有功能、数据结构、代码位置。

---

## 一、面板整体架构

当用户在 Scene 列表中点击选中一个 Character 对象时，左下角属性区渲染 `InspectedElement` 组件，它由 **7 个 Tab 标签 + 对应 Panel** 组成。

```
Inspector Panel
├── 标题栏："{角色名} 特性："（可点击修改名称）
└── Tabs（7 个图标按钮 + 对应内容区）
    ├── Tab 1：icon-tab-parameters  → GeneralInspector（通用参数）
    ├── Tab 2：icon-tab-hand        → HandPresetsEditor（手部姿势预设）
    ├── Tab 3：icon-tab-pose        → PosePresetsInspector（全身姿势预设）
    ├── Tab 4：icon-tab-model       → ModelInspector（模型选择）
    ├── Tab 5：icon-tab-attachable  → AttachableInspector（附件）
    ├── Tab 6：icon-tab-emotions    → EmotionInspector（面部表情）
    └── Tab 7：icon-tab-hair        → HairInspector（头发）
```

**核心入口文件：**
```
src/js/shot-generator/components/InspectedElement/index.js
```

**Tab 系统实现：**
```
src/js/shot-generator/components/Tabs/index.js
```
使用 React Context 管理当前激活 Tab，`<Tab>` 为图标容器，`<Panel>` 为内容容器。

**图标资源路径：**
```
src/img/shot-generator/icon-tab-parameters.svg
src/img/shot-generator/icon-tab-hand.svg
src/img/shot-generator/icon-tab-pose.svg
src/img/shot-generator/icon-tab-model.svg
src/img/shot-generator/icon-tab-attachable.svg
src/img/shot-generator/icon-tab-emotions.svg
src/img/shot-generator/icon-tab-hair.svg
```

---

## 二、Tab 1：通用参数（General Inspector）

**图标：** 参数列表图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/GeneralInspector/Character.js
```

### 提供的参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| X | 数字滑块 | -30 ~ 30 | 世界坐标 X 轴位置（单位：米） |
| Y | 数字滑块 | -30 ~ 30 | 世界坐标 Y 轴位置（单位：米） |
| Z | 数字滑块 | -30 ~ 30 | 世界坐标 Z 轴位置（单位：米） |
| 旋转 | 数字滑块 | -180° ~ 180° | Y 轴旋转角度（弧度存储，度显示） |
| 高度 | 数字滑块 | 因角色类型而异 | 内置角色：英尺显示；自定义模型：缩放比例 |
| 头部 | 数字滑块 | 80% ~ 120% | 头部缩放（headScale）百分比 |
| 淡色 | 颜色选择器 | — | 皮肤色调（tintColor） |
| 变形目标 | 滑块组 | 0~100% | mesomorphic（muscular）/ ectomorphic（skinny）/ endomorphic（obese）|
| 落地按钮 | 按钮 | — | 触发 IPC `shot-generator:object:drops` 使角色落到地面 |

### 角色高度范围（meters）

```javascript
// src/js/shot-generator/components/InspectedElement/GeneralInspector/Character.js
const CHARACTER_HEIGHT_RANGE = {
  character: { min: 1.4732, max: 2.1336 },  // adult / teen
  child:     { min: 1.003,  max: 1.384  },
  baby:      { min: 0.492,  max: 0.94   }
}
```

### CharacterPresetEditor（预设选择器，位于通用参数顶部）

**组件文件：**
```
src/js/shot-generator/components/InspectedElement/CharacterPresetEditor/index.js
```

"预设" 下拉行（即截图中 `预设 --- +` 那一行）：
- 从 `state.presets.characters` 中读取已保存的角色预设
- 点击 `+` 保存当前人物（模型、高度、头部比例、色调、体型变形、附件）为新预设
- 选择某个预设时会：重置骨架 → 替换模型 → 恢复附件列表

**用户预设存储路径（Mac）：**
```
~/Library/Application Support/Storyboarder/presets/characters.json
```

---

## 三、Tab 2：手部姿势预设（Hand Poses）

**图标：** 手掌图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/index.js
src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/HandPresetsEditorItem.js
```

### 功能说明

| 功能 | 说明 |
|------|------|
| 搜索框 | 按名称/关键词实时过滤手部预设 |
| `+` 按钮 | 保存当前手骨骼状态为新预设，弹出命名框，并选择存为左手/右手 |
| 手型选择器 | Select 下拉，选择「Left Hand / Right Hand / Both Hands」决定应用到哪只手 |
| 预设缩略图网格 | 4 列展示，每格 68×100px 缩略图 + 名称 |
| 右键/长按预设 | 删除预设（内置预设不可删除） |

### 数据结构

```json
{
  "EA5EE721-F78E-405A-BBA8-F466D38B60D6": {
    "id": "EA5EE721-F78E-405A-BBA8-F466D38B60D6",
    "name": "Relaxed",
    "keywords": "Relaxed",
    "priority": 0,
    "state": {
      "handSkeleton": {
        "RightHandThumb1": { "rotation": { "x": 0.18, "y": -0.18, "z": -0.50 } },
        "RightHandThumb2": { "rotation": { "x": -0.005, "y": 0.0, "z": 0.0 } },
        "RightHandIndex1": { "rotation": { "x": ..., "y": ..., "z": ... } }
      }
    }
  }
}
```

- 骨骼名以 `RightHand` 或 `LeftHand` 开头区分左右手
- 旋转值为弧度（radians）

### 内置手势预设列表（32 个）

```
Relaxed, Natural Rest, Flat Natural, Flat Spread, Flat Together,
Open Curled, Point, Point Relaxed, OK, Peace, Peace 2, Middle Finger,
Thumbs up, Fist, Rock, Spiderman, Middle Together Relaxed, Pinky Up,
Telephone, Scratch Claw, Reach, Hold Pistol Gun, Hold Pencil,
Hold Sphere, Hold Stick, Hold Tiny Rice, Hold Coin, Hold Small Object,
Three 3, Four 4, Claw Extreme, Hold Palm
```

### 数据文件路径

| 类型 | 路径 |
|------|------|
| 内置默认预设 | `src/js/shared/reducers/shot-generator-presets/hand-poses.json` |
| 用户自定义预设（Mac） | `~/Library/Application Support/Storyboarder/presets/hand-poses.json` |
| 缩略图缓存 | `~/Library/Application Support/Storyboarder/presets/hand-poses/{id}.jpg` |

### 应用逻辑（关键代码）

```javascript
// 保存预设时按 handName 过滤骨骼：
for(let i = 0; i < originalSkeleton.length; i++) {
  let key = originalSkeleton[i].name
  if(key.includes(handName) && key !== handName) {
    handSkeleton[key] = { rotation: { x, y, z } }
  }
}

// 应用时触发 Redux action：
updateObject(id, { handPosePresetId: newPreset.id })
```

---

## 四、Tab 3：全身姿势预设（Pose Presets）

**图标：** 骨骼姿势图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js
src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js
```

### 功能说明

| 功能 | 说明 |
|------|------|
| 搜索框 | 按名称/关键词实时过滤姿势 |
| `+` 按钮 | 保存当前全身骨骼状态为新预设，弹出命名框 |
| **镜像姿势** | 对当前骨骼做左右镜像（四元数 x、w 取反），用于生成对称姿势 |
| 预设缩略图网格 | 4 列展示，每格 68×100px 缩略图（ThumbnailRenderer 渲染）|
| 删除预设 | 内置默认预设不可删除，用户自定义可删 |

### 镜像算法

```javascript
// src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js
let mirroredQuat = new THREE.Quaternion().setFromEuler(
  new THREE.Euler(boneRot.x, boneRot.y, boneRot.z)
)
mirroredQuat.x *= -1
mirroredQuat.w *= -1

// 同时交换左右骨骼名称：Left ↔ Right
boneName = boneName.replace(currentSide, oppositeSide)
```

### 数据结构

```json
{
  "AE56DD1E-3F6F-4A74-B247-C8A6E3EB8FC0": {
    "id": "AE56DD1E-3F6F-4A74-B247-C8A6E3EB8FC0",
    "name": "Default Pose",
    "keywords": "Default Pose",
    "priority": 0,
    "state": {
      "skeleton": {
        "Hips": {
          "rotation": { "x": 0.0, "y": 0.0, "z": 0.0 },
          "position": { "x": 0.0, "y": 0.0, "z": 0.0 }
        },
        "Spine": { "rotation": { ... }, "position": { ... } }
      }
    }
  }
}
```

### 内置姿势预设（共 342 个，部分示例）

```
站立类：stand, stand 2, stand 3, wide stance, explaining, walking
坐姿类：sit chair, sit cross legged, sit on floor, sit hug knees
动作类：walk, run, jump, dive, freefall, kneel, squat, crouch inspect
互动类：aim gun, carrying box, open door, point, hi, shrug
倒地类：laying on back, laying on front, fall 1, fall 2, kneeling hurt
格斗类：kick, punch, defend raised arms, body check, hit from behind
特殊类：riding a bike, swimming, skateboarding, push back explosion
...共 342 个
```

### 数据文件路径

| 类型 | 路径 |
|------|------|
| 内置默认预设 | `src/js/shared/reducers/shot-generator-presets/poses.json` |
| 用户自定义预设（Mac） | `~/Library/Application Support/Storyboarder/presets/poses.json` |
| 缩略图缓存 | `~/Library/Application Support/Storyboarder/presets/poses/{id}.jpg` |

### 应用逻辑

```javascript
// 应用预设时通过 Redux action 更新骨骼：
updateObject(id, { posePresetId: newPreset.id, skeleton: preset.state.skeleton })

// 创建新预设时同时上报到服务器（姿势采集）：
request.post('https://storyboarders.com/api/create_pose', { form: { name, json: skeleton, ... } })
```

---

## 五、Tab 4：模型选择（Model Inspector）

**图标：** 人形模型图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/ModelInspector/index.js
src/js/shot-generator/components/InspectedElement/ModelInspectorItem/index.js
```

### 功能说明

| 功能 | 说明 |
|------|------|
| 搜索框 | 按模型名称/关键词过滤 |
| 内置模型网格 | 从 `state.models` 中过滤 `type === 'character'` 的模型展示 |
| 自定义模型 | 通过 FileInput 从本地加载 `.glb` 文件，自动复制到项目目录 |
| 切换模型 | 选择新模型时自动重置骨架（`updateCharacterIkSkeleton({id, skeleton:[]})`） |

### 模型数据来源

```
src/js/shared/reducers/shot-generator.js  →  initialState.models
```

内置模型包含：adult-male, adult-female, teen-male, teen-female, child, baby 等

**模型资源文件（`.glb`）存储路径：**
```
src/data/shot-generator/volumes/
```

---

## 六、Tab 5：附件（Attachable Inspector）

**图标：** 附件/眼镜图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js
```

### 功能说明

- 列出所有 `type === 'attachable'` 且 `attachableType !== 'hair'` 的模型（帽子、眼镜、道具等）
- 点击附件 → 弹出 `HandSelectionModal` 选择绑定到哪根骨骼
- 支持自定义附件（本地 `.glb`）
- 选中后附件作为独立 `sceneObject` 创建，通过 `attachToId` 绑定到角色

### Redux Action

```javascript
createObject({
  id, type: 'attachable', attachableType: 'prop',
  bindBone: 'RightHand',  // 绑定骨骼名
  attachToId: characterId
})
```

---

## 七、Tab 6：面部表情（Emotion Inspector）

**图标：** 表情/面具图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js
```

### 功能说明

| 功能 | 说明 |
|------|------|
| 搜索框 | 按名称/关键词过滤表情 |
| `+` 按钮 | 导入自定义面部贴图（PNG 文件），创建新表情预设 |
| 预设网格 | 展示缩略图，点击应用到当前角色 |
| 删除 | 内置表情不可删除 |

### 内置表情预设（6 个）

```
Joker, Angry, Cry, Dead, Garold, Neutral
```

### 数据结构

```json
{
  "27934E27-CB0C-40EF-B72C-5A939DA3BFB3": {
    "id": "27934E27-CB0C-40EF-B72C-5A939DA3BFB3",
    "name": "Joker",
    "keywords": "Joker",
    "priority": 0
    // 图片文件另存于 userData/presets/emotions/{id}.png
  }
}
```

### 数据文件路径

| 类型 | 路径 |
|------|------|
| 内置默认预设 | `src/js/shared/reducers/shot-generator-presets/emotions.json` |
| 用户自定义预设（Mac） | `~/Library/Application Support/Storyboarder/presets/emotions.json` |
| 表情贴图 | `~/Library/Application Support/Storyboarder/presets/emotions/{id}.png` |

**帮助文档链接：** `https://github.com/wonderunit/storyboarder/wiki/Creating-Emotions-for-Characters-in-Shot-Generator`

---

## 八、Tab 7：头发（Hair Inspector）

**图标：** 帽子/头发图标  
**组件文件：**
```
src/js/shot-generator/components/InspectedElement/HairInspector/index.js
```

### 功能说明

- 列出所有 `attachableType === 'hair'` 的模型
- 点击应用后创建附件对象，自动绑定到 `Head` 骨骼
- 支持自定义头发模型（本地 `.glb`）

### 头发附件绑定常量

```javascript
const USER_MODEL_HAIR_POSITION = { x: -0.0013, y: 0.15, z: 0.014 }

// 创建时自动设置：
{ type: 'attachable', attachableType: 'hair', bindBone: 'Head' }
```

---

## 九、骨骼 IK 系统（Bone Inspector）

当在 3D 视图中点击选中某根骨骼时，GeneralInspector 底部会额外显示 `BoneInspector`。

**组件文件：**
```
src/js/shot-generator/components/InspectedElement/BoneInspector/index.js
```

### 功能

- 显示当前选中骨骼名称
- 提供 X / Y / Z 旋转滑块（-180° ~ 180°，角度制显示，弧度存储）
- 实时更新骨骼旋转 → 驱动 IK 系统

### Redux Action

```javascript
// 单根骨骼旋转：
updateCharacterSkeleton({ id: characterId, name: boneName, rotation: { x, y, z } })

// 批量更新整个骨架（应用预设/镜像时）：
updateCharacterIkSkeleton({ id: characterId, skeleton: [...] })

// IK 极点目标：
updateCharacterPoleTargets({ id: characterId, poleTargets: {...} })
```

**IK 系统核心文件：**
```
src/js/shared/IK/SGIkHelper.js  （~350行，IK 单例，管理 Ragdoll 和控制点）
```

---

## 十、预设持久化系统

**统一入口文件：**
```
src/js/shared/store/presetsStorage.js
```

### 存储路径汇总（Mac）

| 预设类型 | userData 路径 |
|----------|--------------|
| 场景预设 | `presets/scenes.json` |
| 角色预设 | `presets/characters.json` |
| 姿势预设 | `presets/poses.json` |
| 手势预设 | `presets/hand-poses.json` |
| 表情预设 | `presets/emotions.json` |

> `userData` 路径（Mac）= `~/Library/Application Support/Storyboarder/`

### 内置默认预设路径（打包在 app 内）

```
src/js/shared/reducers/shot-generator-presets/poses.json        （342 个姿势）
src/js/shared/reducers/shot-generator-presets/hand-poses.json   （32 个手势）
src/js/shared/reducers/shot-generator-presets/emotions.json     （6 个表情）
```

### 注意：内置预设保护机制

用户预设保存时会过滤掉内置预设（denylist），确保内置预设不被意外覆盖：

```javascript
let denylist = Object.keys(defaultPosePresets)  // 内置预设 ID 列表
let filteredPoses = Object.values(state.presets.poses)
  .filter(pose => denylist.includes(pose.id) === false)
  .reduce((coll, pose) => { coll[pose.id] = pose; return coll }, {})
presetsStorage.savePosePresets({ poses: filteredPoses })
```

---

## 十一、缩略图渲染系统

**工具类文件：**
```
src/js/shot-generator/utils/ThumbnailRenderer.js
```

- 使用独立的离屏 `WebGLRenderer` 渲染 68×100px（实际 136×200px @2x）的缩略图
- 应用 `OutlineEffect` 渲染卡通风格轮廓线
- 缩略图以 base64 JPEG 格式保存到 userData 目录
- 每次打开预设列表时懒加载渲染，已有缓存则直接读取

### 渲染参数

```javascript
camera.position = { y: 1, z: 2 }
camera.fov = 75
outlineEffect.defaultThickness = 0.018
background = #3e4043
```

---

## 十二、Redux State 结构（人物相关字段）

```javascript
// 单个 Character sceneObject 的 state 字段：
{
  id: "uuid",
  type: "character",
  model: "adult-female",            // 模型 ID
  displayName: "Adult-female 1",
  x: -0.06, y: 0.93, z: 0.00,      // 世界坐标（米）
  rotation: 0.017,                  // Y 轴旋转（弧度）
  height: 1.65,                     // 身高（米）
  headScale: 1.0,                   // 头部缩放
  tintColor: "#ffffff",             // 皮肤色调
  morphTargets: {
    mesomorphic: 0, ectomorphic: 0, endomorphic: 0
  },
  posePresetId: "uuid | null",      // 当前应用的姿势预设 ID
  skeleton: {                       // 当前骨骼旋转状态
    "Hips": {
      id: "uuid", name: "Hips",
      rotation: { x: 0, y: 0, z: 0 },
      position: { x: 0, y: 0, z: 0 }
    },
    ...
  },
  handPosePresetId: "uuid | null",  // 当前手部预设 ID
  characterPresetId: "uuid | null", // 当前角色外观预设 ID
  emotion: "uuid | null",           // 当前表情预设 ID（文件名）
  visible: true,
  locked: false
}
```

---

## 十三、FlowCanvas 移植要点

### 状态管理替换

| 原始（Electron + Redux + electron-redux） | FlowCanvas（建议） |
|---|---|
| `state.presets.poses` | Zustand store 中的 `posePresets` |
| `presetsStorage.savePosePresets()` | 调用后端 API 或 localStorage |
| `updateCharacterIkSkeleton` Redux action | Zustand `updateIkSkeleton(id, skeleton)` |
| `electron.app.getPath('userData')` | 后端文件存储 API |

### IPC 依赖替换

原始代码中部分功能通过 Electron IPC 触发：
```javascript
ipcRenderer.send('shot-generator:object:drops')  // 落地功能
```
移植时需替换为 FlowCanvas 自身的事件系统或直接调用 Three.js 物理计算。

### 缩略图渲染

`ThumbnailRenderer` 可直接移植到 Web 环境（无 Electron 依赖），只需要去掉 `@electron/remote` 相关引用，改为直接调用文件存储 API。

### 预设数据迁移

三个内置预设 JSON 文件（poses / hand-poses / emotions）可直接复制到 FlowCanvas 项目中作为静态资源，通过 `fetch` 或 `import` 加载即可。

---

## 十四、关键文件一览表

| 文件路径 | 职责 |
|----------|------|
| `src/js/shot-generator/components/InspectedElement/index.js` | 7 个 Tab 的根容器，路由到子检查器 |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Character.js` | 通用参数（位置、旋转、身高、皮肤、体型变形） |
| `src/js/shot-generator/components/InspectedElement/CharacterPresetEditor/index.js` | 角色外观预设保存/加载 |
| `src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/index.js` | 手部姿势预设 UI 和逻辑 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | 全身姿势预设 UI、镜像逻辑 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | 单个姿势预设项，含缩略图渲染 |
| `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js` | 模型选择器 |
| `src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js` | 附件（道具/帽子/眼镜） |
| `src/js/shot-generator/components/InspectedElement/HairInspector/index.js` | 头发模型 |
| `src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js` | 面部表情 |
| `src/js/shot-generator/components/InspectedElement/BoneInspector/index.js` | 单根骨骼 X/Y/Z 微调 |
| `src/js/shot-generator/utils/ThumbnailRenderer.js` | 预设缩略图离屏渲染器 |
| `src/js/shot-generator/utils/InspectorElementsSettings.js` | UI 常量（NUM_COLS=4, ITEM_HEIGHT=132 等） |
| `src/js/shared/reducers/shot-generator.js` | 所有 Redux actions 和 selectors（~1500 行） |
| `src/js/shared/store/presetsStorage.js` | 预设读写到 userData 文件系统 |
| `src/js/shared/reducers/shot-generator-presets/poses.json` | 342 个内置全身姿势数据 |
| `src/js/shared/reducers/shot-generator-presets/hand-poses.json` | 32 个内置手部姿势数据 |
| `src/js/shared/reducers/shot-generator-presets/emotions.json` | 6 个内置表情预设索引 |
| `src/js/shared/IK/SGIkHelper.js` | IK 系统单例，Ragdoll 和控制点管理 |
