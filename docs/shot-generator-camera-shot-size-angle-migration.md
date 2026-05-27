# Shot Generator 相机景别与镜头角度预设功能移植分析

本文档说明 Storyboarder Shot Generator 底部相机工具栏中 `Shot Size`（景别）和 `Camera Angle`（镜头角度）两个预设功能的实现方式、核心代码位置、数据流和移植到 FlowCanvas 时的建议方案。

截图中红框标出的两个控件不是普通文本输入，而是自动构图预设：

- `Shot Size`：根据人物骨骼包围盒调整相机距离，实现特写、中景、全景等景别。
- `Camera Angle`：在当前构图目标基础上调整相机高度 / 俯仰方向，实现鸟瞰、高角度、平视、低角度、虫视等镜头角度。

## 1. 功能入口

底部相机工具栏入口：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

页面布局入口：

- `src/js/shot-generator/components/Editor/index.js`

`CameraPanelInspector` 位于主相机视图下方，负责相机 roll、pan/tilt、move、elevate、lens，以及截图中的 `Shot Size` 和 `Camera Angle`。

两个控件渲染代码：

```jsx
<div className="camera-item shots" {...shotsizeTooltipEvents}>
  <div className="select">
    <Select
      label="Shot Size"
      value={ shotSizes.find(option => option.value === currentShotSize) }
      options={ shotSizes }
      onSetValue={ (item) => onSetShot({ size: item.value, angle: shotInfo.angle }) }/>
  </div>
  <div className="select">
    <Select
      label="Camera Angle"
      value={ shotAngles.find(option => option.value === currentShotAngle) }
      options={ shotAngles }
      onSetValue={ (item) => onSetShot({ size: shotInfo.size, angle: item.value }) }/>
  </div>
</div>
```

`Select` 组件封装了 `react-select`：

- `src/js/shot-generator/components/Select/index.js`

```js
<ReactSelect
  options={ options }
  value={ value }
  placeholder={ label }
  onChange={ callbackRef.current }
  isSearchable={ false }
  menuPlacement="auto"
  menuPosition="fixed"
/>
```

## 2. UI 状态与 Redux 状态

### 2.1 数据来源

`CameraPanelInspector` 从 Redux 读取：

```js
state => ({
  activeCamera: getSceneObjects(state)[getActiveCamera(state)],
  cameraShots: state.cameraShots,
  mainViewCamera: state.mainViewCamera,
})
```

其中：

- `activeCamera` 是当前激活的 camera scene object。
- `cameraShots` 是独立于 `sceneObjects` 的自动构图预设状态。
- `cameraShots[activeCamera.id]` 保存当前相机选择的景别和镜头角度。

### 2.2 本地 UI 状态

组件内部维护两个本地 state，用于让下拉框显示当前值：

```js
const shotInfo = cameraShots[activeCamera.id] || {}
const [currentShotSize, setCurrentShotSize] = useState(shotInfo.size)
const [currentShotAngle, setCurrentShotAngle] = useState(shotInfo.angle)
```

当 Redux 状态变化或 active camera 切换时同步：

```js
useEffect(() => {
  setCurrentShotSize(shotInfo.size)
}, [shotInfo.size, activeCamera])

useEffect(() => {
  setCurrentShotAngle(shotInfo.angle)
}, [shotInfo.angle, activeCamera])
```

### 2.3 写入 action

下拉框选择后调用：

```js
const onSetShot = ({size, angle}) => {
  setCameraShot(activeCamera.id, {size, angle})
}
```

Action creator 定义在：

- `src/js/shared/reducers/shot-generator.js`

```js
setCameraShot: (cameraId, values) => ({
  type: 'SET_CAMERA_SHOT',
  payload: { cameraId, ...values }
})
```

Reducer：

```js
const getCameraShot = (draft, cameraId) => {
  if (!draft[cameraId]) {
    draft[cameraId] = {
      size: null,
      angle: null,
      cameraId: cameraId
    }
  }
  return draft[cameraId]
}

case 'SET_CAMERA_SHOT':
  const camera = getCameraShot(draft, action.payload.cameraId)
  camera.size = action.payload.size || camera.size
  camera.angle = action.payload.angle || camera.angle
  camera.character = action.payload.character
  return
```

注意：

- `size` 和 `angle` 用 `||` 合并，因此不能用空字符串清空。
- `character` 每次都会被赋值，未传时会变成 `undefined`。
- 创建 camera 时会初始化 `cameraShots[cameraId]`。
- 该 reducer 没有处理 `LOAD_SCENE`，这意味着 `cameraShots` 是当前会话中的辅助状态，移植时建议明确是否要持久化。

## 3. 预设枚举

景别和镜头角度定义在：

- `src/js/shot-generator/utils/cameraUtils.js`

### 3.1 ShotSizes

```js
const ShotSizes = {
  EXTREME_CLOSE_UP: 'Extremely close up',
  VERY_CLOSE_UP: 'Very close up',
  CLOSE_UP: 'Close up',
  MEDIUM_CLOSE_UP: 'Medium close up',
  BUST: 'Bust',
  MEDIUM: 'Medium',
  MEDIUM_LONG: 'Medium long',
  LONG: 'Long',
  EXTREME_LONG: 'Extremely long',
  ESTABLISHING: 'Establishing',
  OTS_LEFT: 'OTS Left',
  OTS_RIGHT: 'OTS Right',
}
```

底部 UI 中展示的选项只包含前 10 个，没有展示 `OTS_LEFT` 和 `OTS_RIGHT`：

```js
const shotSizes = [
  { value: ShotSizes.EXTREME_CLOSE_UP,  label: "Extreme Close Up" },
  { value: ShotSizes.VERY_CLOSE_UP,     label: "Very Close Up" },
  { value: ShotSizes.CLOSE_UP,          label: "Close Up" },
  { value: ShotSizes.MEDIUM_CLOSE_UP,   label: "Medium Close Up" },
  { value: ShotSizes.BUST,              label: "Bust" },
  { value: ShotSizes.MEDIUM,            label: "Medium Shot" },
  { value: ShotSizes.MEDIUM_LONG,       label: "Medium Long Shot" },
  { value: ShotSizes.LONG,              label: "Long Shot / Wide" },
  { value: ShotSizes.EXTREME_LONG,      label: "Extreme Long Shot" },
  { value: ShotSizes.ESTABLISHING,      label: "Establishing Shot" }
]
```

### 3.2 ShotAngles

```js
const ShotAngles = {
  BIRDS_EYE: 'Birds',
  HIGH: 'High',
  EYE: 'Eye',
  LOW: 'Low',
  WORMS_EYE: 'Worms'
}
```

UI 展示：

```js
const shotAngles = [
  { value: ShotAngles.BIRDS_EYE, label: "Bird's Eye" },
  { value: ShotAngles.HIGH,      label: "High" },
  { value: ShotAngles.EYE,       label: "Eye" },
  { value: ShotAngles.LOW,       label: "Low" },
  { value: ShotAngles.WORMS_EYE, label: "Worm's Eye" }
]
```

角度对应的弧度值：

```js
const ShotAnglesInfo = {
  [ShotAngles.BIRDS_EYE]: -30 * THREE.Math.DEG2RAD,
  [ShotAngles.HIGH]: -15 * THREE.Math.DEG2RAD,
  [ShotAngles.EYE]: 0,
  [ShotAngles.LOW]: 30 * THREE.Math.DEG2RAD,
  [ShotAngles.WORMS_EYE]: 45 * THREE.Math.DEG2RAD
}
```

这些值不是直接写入 camera.tilt，而是在自动构图时用来绕横向轴旋转“相机到目标点”的方向向量。

## 4. 景别如何计算

核心函数：

- `getShotBox(character, shotType)`
- `getShotInfo({ selected, characters, shotSize, camera })`
- `clampCameraToBox({ camera, box, direction })`
- `setShot({ camera, characters, selected, updateObject, shotSize, shotAngle })`

均位于：

- `src/js/shot-generator/utils/cameraUtils.js`

### 4.1 ShotSizesInfo

`ShotSizesInfo` 定义每种景别应该参考人物哪些骨骼，以及是否裁切上下边界：

```js
const ShotSizesInfo = {
  [ShotSizes.EXTREME_CLOSE_UP]: {
    bones: ['Head', 'leaf', 'Neck'],
    yReduction: [0.08, 0.0]
  },
  [ShotSizes.MEDIUM]: {
    bones: [
      'Head', 'Neck', 'leaf',
      'RightShoulder', 'LeftShoulder',
      'Hips'
    ],
    yReduction: [0, -0.04]
  },
  [ShotSizes.LONG]: {
    bones: [
      'Head', 'Neck', 'leaf',
      'RightShoulder', 'LeftShoulder',
      'Hips',
      'LeftUpLeg', 'RightUpLeg',
      'LeftLeg', 'RightLeg',
      'leaf011', 'leaf012'
    ],
    yReduction: [0, -0.04]
  },
  [ShotSizes.EXTREME_LONG]: {
    bones: [
      'Head', 'Neck', 'leaf',
      'RightShoulder', 'LeftShoulder',
      'Hips',
      'LeftUpLeg', 'RightUpLeg',
      'LeftLeg', 'RightLeg',
      'leaf011', 'leaf012'
    ],
    relativeToScreen: 1.0 / 3.0
  }
}
```

规律：

- 特写类只取头、脖子附近骨骼。
- 中景开始包含肩膀、脊柱、髋部。
- 全景包含腿部和脚部 leaf 骨骼。
- `EXTREME_LONG` 使用 `relativeToScreen`，让角色高度约占画面高度的 1/3。
- `ESTABLISHING` 不使用单人骨骼定义，而是把所有人物扩展进同一个 box。

### 4.2 人物骨骼包围盒

`getShotBox` 会从角色的 `SkinnedMesh.skeleton.bones` 中筛选目标骨骼：

```js
let skinnedMesh = character.getObjectByProperty("type", "SkinnedMesh")
let bones = skinnedMesh.skeleton.bones.filter(
  bone => shotInfo.bones.indexOf(bone.name) !== -1
)
```

每根 bone 用 `matrixWorld` 计算世界坐标：

```js
const getBoneStartEndPos = (bone) => {
  let boneWorldPosition = new THREE.Vector3().setFromMatrixPosition(bone.matrixWorld)
  return {
    start: boneWorldPosition,
    end: boneWorldPosition.clone().add(bone.position)
  }
}
```

然后扩展 box：

```js
bones.forEach((bone) => {
  let boneInfo = getBoneStartEndPos(bone)
  box.expandByPoint(boneInfo.start)
  box.expandByPoint(boneInfo.end)
})
```

再按景别规则修正 box 高度：

```js
if (shotInfo.yReduction) {
  box.min.y += shotInfo.yReduction[0]
  box.max.y -= shotInfo.yReduction[shotInfo.yReduction.length - 1]
} else if (shotInfo.relativeToScreen) {
  let size = new THREE.Vector3()
  box.getSize(size)
  size.y *= 1.0 / shotInfo.relativeToScreen
  box.min.y -= size.y * 0.5
  box.max.y += size.y * 0.5
}
```

### 4.3 根据包围盒调整相机距离

`clampCameraToBox` 根据包围球半径和相机 fov 计算相机需要离目标多远：

```js
let sphere = new THREE.Sphere()
box.getBoundingSphere(sphere)

let h = sphere.radius / Math.tan(camera.fov / 2 * Math.PI / 180.0)
let newPos = new THREE.Vector3().addVectors(
  sphere.center,
  direction.clone().setLength(h)
)
```

返回：

```js
{
  position: newPos.clone(),
  target: sphere.center.clone()
}
```

也就是说，Shot Size 的核心效果是：

```text
根据所选景别取角色部分骨骼
-> 得到世界空间包围盒
-> 包围盒转包围球
-> 根据 camera.fov 计算相机距离
-> 沿当前相机视线反方向放置相机
-> lookAt 包围盒中心
```

## 5. 镜头角度如何计算

景别先给出构图目标和基础相机位置。镜头角度再在这个基础上调整相机位置。

核心代码：

```js
if (ShotAnglesInfo[shotAngle] !== undefined) {
  let currentDistance = clampedInfo.position.distanceTo(clampedInfo.target)

  direction.y = 0
  direction.normalize()
  let mainAxis = new THREE.Vector3().crossVectors(camera.up, direction)

  let quaternion = new THREE.Quaternion()
  quaternion.setFromAxisAngle(mainAxis, -ShotAnglesInfo[shotAngle])

  direction.applyQuaternion(quaternion)
  direction.setLength(currentDistance)

  clampedInfo.position.copy(clampedInfo.target).sub(direction)
}
```

解释：

- `clampedInfo.target` 是被构图的人物 / 人物群中心。
- `currentDistance` 保持相机到目标点的距离不变。
- `direction.y = 0` 把当前视线投影到水平面。
- `mainAxis = camera.up x direction` 得到绕目标抬高 / 降低相机的横向旋转轴。
- 根据 `ShotAnglesInfo` 的角度旋转 direction。
- 用旋转后的 direction 重新计算 camera position。

然后防止相机进入地下：

```js
if (clampedInfo.position.y < 0) {
  clampedInfo.position.sub(direction.clone().setY(0).setLength(clampedInfo.position.y))
  clampedInfo.position.y = 0
}
```

最后：

```js
camera.position.copy(clampedInfo.position)
camera.lookAt(clampedInfo.target)
camera.updateMatrixWorld(true)
```

并把 Three.js camera 转回 Shot Generator 的 camera scene object：

```js
let rot = new THREE.Euler().setFromQuaternion(camera.quaternion, "YXZ")
updateObject(camera.userData.id, {
  x: camera.position.x,
  y: camera.position.z,
  z: camera.position.y,
  rotation: rot.y,
  roll: rot.z,
  tilt: rot.x
})
```

注意坐标映射：

```text
Three.js position.x -> camera.x
Three.js position.z -> camera.y
Three.js position.y -> camera.z
Euler Y -> camera.rotation
Euler Z -> camera.roll
Euler X -> camera.tilt
```

## 6. 实际触发自动构图的位置

选择下拉框后，只是写入 `cameraShots`。真正移动相机的位置在：

- `src/js/shot-generator/SceneManagerR3fLarge.js`

核心 effect：

```js
useEffect(() => {
  if(!selectedCharacters.current) return
  let selected = scene.children[0].children.find(
    obj => selectedCharacters.current.indexOf(obj.userData.id) >= 0
  )
  let characters = scene.children[0].children.filter(
    obj => obj.userData.type === "character"
  )
  if (characters.length) {
    let keys = Object.keys(cameraShots)
    for(let i = 0; i < keys.length; i++ ) {
      let key = keys[i]
      if(cameraShots[key].character) {
        selected = scene.__interaction.filter(
          object => object.userData.id === cameraShots[key].character
        )[0]
      }
      if((!cameraShots[key].size && !cameraShots[key].angle) || camera.userData.id !== cameraShots[key].cameraId) continue
      setShot({
        camera,
        characters,
        selected,
        updateObject,
        shotSize: cameraShots[key].size,
        shotAngle: cameraShots[key].angle
      })
    }
  }
}, [cameraShots])
```

运行逻辑：

```text
cameraShots 改变
-> 找到当前选中的 character，或 cameraShots[key].character 指定的 character
-> 收集场景中所有 character
-> 只处理当前 Three.js camera 对应的 cameraShots 记录
-> 调用 setShot()
-> setShot 计算并移动 Three.js camera
-> updateObject 写回 camera scene object
```

因此这两个下拉框的本质是：

```text
UI 选择景别 / 角度
-> 更新 cameraShots
-> SceneManagerR3fLarge 监听 cameraShots
-> setShot 自动重算相机位置和朝向
-> updateObject 写回 active camera
```

## 7. 目标人物选择规则

`SceneManagerR3fLarge` 会优先使用当前选中的 character：

```js
let selected = scene.children[0].children.find(
  obj => selectedCharacters.current.indexOf(obj.userData.id) >= 0
)
```

如果 `cameraShots[key].character` 存在，则用它覆盖：

```js
if(cameraShots[key].character) {
  selected = scene.__interaction.filter(
    object => object.userData.id === cameraShots[key].character
  )[0]
}
```

如果没有明确 selected，`setShot()` 内会选择离相机最近且方向合理的角色：

```js
selected: selected || getClosestCharacter(characters, camera)
```

`getClosestCharacter` 会综合：

- 角色是否在相机 frustum 中
- 角色朝向和相机方向 dot 值
- 角色到相机距离

找不到时回退到 `characters[0]`。

## 8. Establishing Shot 特殊处理

`ShotSizes.ESTABLISHING` 不只看一个角色，而是把所有角色都纳入构图：

```js
if (shotSize === ShotSizes.ESTABLISHING) {
  for (let i = 0; i < characters.length; i++) {
    box.expandByObject(characters[i])
  }
}
```

如果有多个角色，还会尝试根据角色之间的位置关系调整 direction：

```js
if (characters.length > 1) {
  direction = new THREE.Vector3()

  for (let i = 0; i < characters.length - 1; i += 2) {
    direction.add(characters[i + 1].position.clone().sub(characters[i].position.clone()))
  }

  direction.divideScalar(characters.length)
  direction = camera.position.clone().sub(direction)
  direction.y = camera.y
}
```

移植时建议重新评估这段逻辑，因为 `camera.y` 在 Three.js camera 上不是标准字段，可能是历史遗留或 bug。FlowCanvas 应使用明确的坐标字段。

## 9. 与 Shot Explorer 的关系

底部还有 `Open Shot Explorer` 按钮：

```js
ipcRenderer.send('shot-generator:show:shot-explorer')
```

Shot Explorer 也复用了同一套 `ShotSizes`、`ShotAngles` 和 `setShot()`：

- `src/js/shot-explorer/ShotMaker.js`
- `src/js/shot-explorer/ShotItem.js`
- `src/js/shot-explorer/ShotExplorerSceneManager.js`

`ShotMaker` 会随机选择景别和角度，生成多个候选镜头：

```js
let randomAngle = ShotAngles[shotAngleKeys[getRandomNumber(shotAngleKeys.length)]]
let randomSize = ShotSizes[shotSizeKeys[getRandomNumber(shotSizeKeys.length - 2)]]
let shot = new ShotItem(randomAngle, randomSize, character)
let box = setShot({camera: cameraCopy, characters, selected:character, shotAngle:shot.angle, shotSize:shot.size})
```

这说明 FlowCanvas 如果后续要做“镜头推荐 / 镜头探索器”，也应该复用同一个自动构图核心函数，而不是另写一套。

## 10. FlowCanvas 移植建议

### 10.1 推荐模块拆分

```text
camera-composition/
├── components/
│   ├── CameraCompositionPanel.tsx
│   └── ShotPresetSelect.tsx
├── state/
│   └── cameraCompositionStore.ts
├── types/
│   └── cameraCompositionTypes.ts
└── utils/
    ├── shotPresets.ts
    ├── shotBox.ts
    ├── shotFraming.ts
    ├── cameraCoordinate.ts
    └── characterTargeting.ts
```

职责：

- `shotPresets.ts`：定义 `ShotSize`、`CameraAngle`、骨骼映射和角度映射。
- `shotBox.ts`：根据人物骨骼计算构图 box。
- `shotFraming.ts`：根据 box、fov、angle 计算 camera transform。
- `cameraCoordinate.ts`：集中处理 FlowCanvas 状态坐标与 Three.js 坐标转换。
- `characterTargeting.ts`：选择当前构图目标角色。
- `CameraCompositionPanel.tsx`：渲染 `Shot Size` 和 `Camera Angle`。

### 10.2 推荐类型

```ts
export type ShotSize =
  | 'extreme-close-up'
  | 'very-close-up'
  | 'close-up'
  | 'medium-close-up'
  | 'bust'
  | 'medium'
  | 'medium-long'
  | 'long'
  | 'extreme-long'
  | 'establishing'

export type CameraAngle =
  | 'birds-eye'
  | 'high'
  | 'eye'
  | 'low'
  | 'worms-eye'

export interface CameraShotPreset {
  cameraId: string
  size: ShotSize | null
  angle: CameraAngle | null
  characterId?: string | null
}

export interface CameraTransformPatch {
  x: number
  y: number
  z: number
  rotation: number
  tilt: number
  roll: number
}
```

### 10.3 推荐 action

```ts
setCameraShotPreset(input: {
  cameraId: string
  size?: ShotSize | null
  angle?: CameraAngle | null
  characterId?: string | null
}): void

applyCameraShotPreset(cameraId: string): void
```

实现建议：

- UI 下拉只调用 `setCameraShotPreset`。
- 状态层或 effect 监听 preset 变化后调用 `applyCameraShotPreset`。
- `applyCameraShotPreset` 通过纯函数计算 camera transform patch，再统一写回 camera object。

### 10.4 推荐计算函数

```ts
export function frameCameraForShot(input: {
  camera: THREE.PerspectiveCamera
  characters: THREE.Object3D[]
  selectedCharacter?: THREE.Object3D | null
  shotSize?: ShotSize | null
  cameraAngle?: CameraAngle | null
}): {
  cameraPosition: THREE.Vector3
  target: THREE.Vector3
  box: THREE.Box3
}
```

该函数应做到：

- 不直接修改 store。
- 可选择是否直接修改 Three.js camera。
- 返回可测试的 position / target / box。

## 11. 需要改进的边界情况

Storyboarder 当前实现能工作，但移植时建议补强以下点。

### 11.1 无人物时

当前主场景 effect 中只有 `characters.length` 时才运行。FlowCanvas 应在 UI 上禁用两个 select，或显示“需要至少一个人物”。

### 11.2 非标准骨骼命名

景别依赖骨骼名，例如：

```text
Head
Neck
leaf
RightShoulder
LeftShoulder
Hips
LeftLeg
RightLeg
leaf011
leaf012
```

FlowCanvas 应通过角色导入阶段的 skeleton mapping 提供标准骨骼名，避免直接依赖模型原始 bone name。

### 11.3 当前相机必须是 PerspectiveCamera

算法依赖：

```js
camera.fov
camera.lookAt()
camera.getWorldDirection()
```

如果 FlowCanvas 支持 orthographic camera，应禁用该功能或单独实现正交构图。

### 11.4 坐标映射

Storyboarder 使用：

```text
state x/y/z = x / ground-z / height-y
Three.js x/y/z = x / height-y / ground-z
```

FlowCanvas 必须集中处理坐标转换，不能把 `x, y: z, z: y` 这种映射散落在 UI 组件中。

### 11.5 状态持久化

Storyboarder 的 `cameraShots` 是独立状态，当前 reducer 没有显式 `LOAD_SCENE` 恢复逻辑。FlowCanvas 需要决定：

- 如果希望打开项目后仍记住下拉框选择，应把 `cameraShotPreset` 持久化到项目数据。
- 如果只把它当作一次性工具，则保存相机 transform 即可，不必保存 preset。

### 11.6 Undo / Redo

当前下拉选择会触发：

```text
SET_CAMERA_SHOT
-> setShot
-> UPDATE_OBJECT camera transform
```

FlowCanvas 应把“选择预设 + 更新相机 transform”作为一次 undoable 操作，避免 undo 时只撤销一半状态。

## 12. 验收标准

### UI 行为

- 底部相机面板显示 `Shot Size` 和 `Camera Angle` 两个选择器。
- 没有 active camera 时不显示或禁用。
- 没有 character 时禁用，并给出提示。
- 切换 active camera 时，下拉框显示该 camera 的当前预设。

### 构图行为

- 选择 `Close Up` 后，相机对准人物头颈区域。
- 选择 `Medium Shot` 后，相机覆盖头、肩、髋部附近。
- 选择 `Long Shot / Wide` 后，相机覆盖完整人物。
- 选择 `Establishing Shot` 后，相机覆盖场景中全部人物。
- 修改镜头焦距后，再选择景别，相机距离会随 fov 正确变化。

### 角度行为

- `Bird's Eye` 相机位于较高俯拍位置。
- `High` 是轻微高角度。
- `Eye` 接近平视。
- `Low` 是低角度仰拍。
- `Worm's Eye` 是更极端的仰拍。
- 相机不会落到地面以下。

### 状态行为

- 选择预设后 camera object 的 `x/y/z/rotation/tilt/roll` 被写回。
- 选择预设后主视图立即更新。
- 俯视预览中的 camera icon 跟随更新。
- Undo / redo 后相机 transform 和预设显示保持一致。

## 13. 最小移植步骤

1. **移植枚举和配置**
   - `ShotSizes`
   - `ShotAngles`
   - `ShotSizesInfo`
   - `ShotAnglesInfo`

2. **移植构图纯函数**
   - `getShotBox`
   - `getShotInfo`
   - `clampCameraToBox`
   - `setShot` 拆成 `computeShotCameraTransform`

3. **接入人物骨骼标准化**
   - 确认 FlowCanvas character runtime object 可访问 `SkinnedMesh.skeleton.bones`。
   - 确认 bone name 已映射到标准名。

4. **接入状态**
   - 新增 `cameraShotPreset` 状态。
   - 新增 `setCameraShotPreset` 和 `applyCameraShotPreset`。

5. **接入 UI**
   - 在相机面板中添加两个 select。
   - 选择后写入 preset 并触发构图。

6. **接入 undo / 持久化**
   - 把预设选择和相机 transform 更新纳入同一个 undo group。
   - 决定是否把 preset 本身保存到项目文件。

## 14. 关键代码清单

```text
src/js/shot-generator/components/CameraPanelInspector/index.js
src/js/shot-generator/components/Select/index.js
src/js/shot-generator/SceneManagerR3fLarge.js
src/js/shot-generator/utils/cameraUtils.js
src/js/shared/reducers/shot-generator.js
src/js/shot-explorer/ShotMaker.js
src/js/shot-explorer/ShotItem.js
```

如果 FlowCanvas 只移植截图中的两个下拉功能，最关键的是：

```text
CameraPanelInspector/index.js 中的 onSetShot 和 select 配置
SceneManagerR3fLarge.js 中监听 cameraShots 后调用 setShot 的 effect
cameraUtils.js 中的 ShotSizes / ShotAngles / setShot
shot-generator.js 中 cameraShotsReducer 和 setCameraShot
```
