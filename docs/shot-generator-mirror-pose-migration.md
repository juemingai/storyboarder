# Shot Generator 镜像姿势功能移植分析

本文档说明 Storyboarder Shot Generator 中“镜像姿势（Mirror pose）”功能的实现方式、代码位置、数据变换规则，以及移植到 FlowCanvas 时建议采用的实现方案。

该功能属于人物姿势编辑系统的一部分，依赖角色当前的 `sceneObject.skeleton`，不直接操作 Three.js 场景中的 bone。它通过生成一份“左右互换 + 旋转镜像”的 skeleton，再调用现有的 `updateCharacterIkSkeleton` action 批量写回角色姿势。

## 1. 功能入口

镜像姿势按钮位于人物属性面板的“姿势预设”页签中。

入口链路：

```text
InspectedElement
-> character 类型对象显示 PosePresetsInspector
-> PosePresetsInspector 渲染 Mirror pose 按钮
-> 点击按钮调用 mirrorSkeleton()
```

相关文件：

- `src/js/shot-generator/components/InspectedElement/index.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js`
- `src/js/locales/zh-CN.json`
- `src/css/shot-generator.css`

`InspectedElement` 中只有选中对象为 `character` 时才显示姿势页签：

```js
const charPoseTab = useMemo(() => {
  if (!isChar(selectedType)) return nullTab

  return {
    tab: <Tab><Icon src='icon-tab-pose'/></Tab>,
    panel: <Panel><PosePresetsInspector/></Panel>
  }
}, [selectedType])
```

按钮渲染位置：

```js
<div className="mirror_button__wrapper">
  <div className="mirror_button" onPointerDown={ mirrorSkeleton }>
    {t('shot-generator.inspector.pose-preset.mirror-pose')}
  </div>
</div>
```

中文文案在：

```json
"mirror-pose": "镜像姿势"
```

样式类在 `src/css/shot-generator.css`：

```css
.mirror_button__wrapper
.mirror_button
```

## 2. 核心实现文件

镜像姿势的完整逻辑集中在：

- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js`

关键函数：

- `limbsSide(limbName)`
- `mirrorSkeleton()`

### 2.1 左右侧判断

`limbsSide` 根据 bone name 中是否包含 `Left` 判断当前侧和目标侧：

```js
const limbsSide = (limbName) => {
  let currentSide, oppositeSide
  if(limbName.includes("Left")) {
    currentSide = "Left"
    oppositeSide = "Right"
  } else {
    currentSide = "Right"
    oppositeSide = "Left"
  }
  return {currentSide, oppositeSide}
}
```

注意：

- 该函数默认“不是 Left 就是 Right”。
- 调用方只在 `key.includes("Left") || key.includes("Right")` 时调用它，因此中轴骨骼不会进入此函数。

### 2.2 镜像主流程

`mirrorSkeleton()` 的运行流程：

```text
读取当前选中 character
-> 获取 sceneObject.skeleton
-> 遍历 skeleton 中所有 bone
-> 将 Euler rotation 转为 Quaternion
-> 修改 quaternion.x 和 quaternion.w 的符号
-> 如果 bone 名包含 Left / Right，则替换为对侧 bone name
-> Quaternion 转回 Euler
-> 生成 oppositeSkeleton 数组
-> updateCharacterIkSkeleton({ id, skeleton: oppositeSkeleton })
```

原实现：

```js
const mirrorSkeleton = () => {
  let sceneObject
  withState((dispatch, state) => {
    sceneObject = getSceneObjects(state)[id]
  })
  let oppositeSkeleton = []
  let originalSkeleton = sceneObject.skeleton
  let keys = Object.keys(originalSkeleton)
  for (let i = 0; i < keys.length; i++) {
    let key = keys[i]
    let boneName = key
    let boneRot = originalSkeleton[key].rotation
    let position = originalSkeleton[key].position
    let mirroredQuat = new THREE.Quaternion().setFromEuler(new THREE.Euler(boneRot.x, boneRot.y, boneRot.z))
    mirroredQuat.x *= -1
    mirroredQuat.w *= -1
    if(key.includes("Left") || key.includes("Right")) {
      let {currentSide, oppositeSide} = limbsSide(boneName)
      boneName = boneName.replace(currentSide, oppositeSide)
    }
    let euler = new THREE.Euler().setFromQuaternion(mirroredQuat)
    oppositeSkeleton.push({
      id: originalSkeleton[boneName].id,
      name: boneName,
      rotation : { x: euler.x, y: euler.y, z: euler.z },
      position: { x: position.x, y: position.y, z: position.z }
    })
  }
  updateCharacterIkSkeleton({id:sceneObject.id, skeleton: oppositeSkeleton})
}
```

## 3. 数据结构与写回方式

### 3.1 输入数据

镜像功能读取当前角色对象：

```js
sceneObject = getSceneObjects(state)[id]
```

使用的字段：

```js
sceneObject.id
sceneObject.skeleton
```

典型 `sceneObject.skeleton`：

```js
{
  LeftArm: {
    name: 'LeftArm',
    rotation: { x, y, z },
    position: { x, y, z },
    quaternion: { x, y, z, w }
  },
  RightArm: {
    name: 'RightArm',
    rotation: { x, y, z },
    position: { x, y, z },
    quaternion: { x, y, z, w }
  },
  Spine: {
    name: 'Spine',
    rotation: { x, y, z }
  }
}
```

### 3.2 输出数据

`mirrorSkeleton()` 输出的是数组，不是对象：

```js
[
  {
    id,
    name: 'RightArm',
    rotation: { x, y, z },
    position: { x, y, z }
  }
]
```

该数组传给：

- `updateCharacterIkSkeleton`
- reducer 中的 `UPDATE_CHARACTER_IK_SKELETON`

Reducer 位置：

- `src/js/shared/reducers/shot-generator.js`

写回逻辑：

```js
case 'UPDATE_CHARACTER_IK_SKELETON':
  for (let bone of action.payload.skeleton) {
    let rotation = bone.rotation
    let position = bone.position
    let quaternion = bone.quaternion

    draft[id].skeleton[bone.name].rotation = { x, y, z }
    draft[id].skeleton[bone.name].position = { x, y, z }
    draft[id].skeleton[bone.name].quaternion = { x, y, z, w }
    draft[id].skeleton[bone.name].name = bone.name
  }
```

镜像功能没有传 `quaternion`，因此 reducer 会保留旧 quaternion 或写入空对象，具体取决于目标 bone 是否已存在。

### 3.3 姿势重新应用

写回后，`Character` 组件监听 `sceneObject.skeleton` 变化，并重新应用骨骼旋转。

相关文件：

- `src/js/shot-generator/components/Three/Character.js`

核心逻辑：

```js
for (let bone of skeleton.bones) {
  let modified = sceneObject.skeleton[bone.name]
  let original = originalSkeleton.getBoneByName(bone.name)
  let state = modified || original

  if (bone.rotation.equals(state.rotation) == false) {
    bone.rotation.setFromVector3(state.rotation)
    bone.updateMatrixWorld()
  }
}
```

因此镜像功能的可见效果不是由 `mirrorSkeleton()` 直接操作 Three.js bone 产生，而是由状态更新后触发 `Character` 重新渲染产生。

## 4. 镜像算法解释

### 4.1 左右骨骼互换

左右骨骼通过名称替换实现：

```js
if(key.includes("Left") || key.includes("Right")) {
  boneName = boneName.replace(currentSide, oppositeSide)
}
```

示例：

```text
LeftArm      -> RightArm
RightForeArm -> LeftForeArm
LeftHand     -> RightHand
RightUpLeg   -> LeftUpLeg
```

中轴骨骼不换名：

```text
Hips
Spine
Spine1
Spine2
Neck
Head
```

### 4.2 Quaternion 镜像

实现方式：

```js
let mirroredQuat = new THREE.Quaternion().setFromEuler(
  new THREE.Euler(boneRot.x, boneRot.y, boneRot.z)
)
mirroredQuat.x *= -1
mirroredQuat.w *= -1
let euler = new THREE.Euler().setFromQuaternion(mirroredQuat)
```

含义：

- 先把当前 bone 的 Euler rotation 转成 quaternion。
- 再通过翻转 `x` 和 `w` 分量得到镜像旋转。
- 最后转回 Euler，继续沿用系统现有的 `rotation: {x,y,z}` 存储格式。

从行为上看，它相当于在角色左右对称面上反射姿势，使左侧动作可以转移到右侧，右侧动作可以转移到左侧，中轴骨骼则获得镜像后的旋转。

### 4.3 position 处理

原实现直接复制原 bone 的 `position`：

```js
position: { x: position.x, y: position.y, z: position.z }
```

它没有对 `position.x` 做取反，也没有在左右骨骼交换时交换 position 来源。

这说明 Storyboarder 的镜像姿势主要依赖 rotation，而不是依赖运行期 position。持久化时 `position` 也会被序列化逻辑移除。

## 5. 与手势镜像功能的关系

项目中还有一套“手势镜像”工具函数：

- `src/js/utils/handSkeletonUtils.js`

实现函数：

- `createdMirroredHand(originalSkeleton, mirroredHandName)`
- `applyChangesToSkeleton(skeletonToApplyTo, changedSkeleton)`
- `getOppositeHandName(handName)`

手势镜像使用了相同的 quaternion 规则：

```js
let mirroredEuler = new THREE.Quaternion().setFromEuler(new THREE.Euler(boneRot.x, boneRot.y, boneRot.z))
mirroredEuler.x *= -1
mirroredEuler.w *= -1
let euler = new THREE.Euler().setFromQuaternion(mirroredEuler)
```

差异：

- 人物姿势镜像处理整个 `sceneObject.skeleton`。
- 手势镜像只处理 `RightHand` / `LeftHand` 下的手部骨骼。
- 手势镜像返回对象，人物姿势镜像返回数组。

FlowCanvas 移植时建议把两者统一抽象为一个通用 skeleton mirror 工具，而不是分别实现两套规则。

## 6. FlowCanvas 移植建议

### 6.1 推荐模块

建议新增独立纯函数模块，避免把镜像算法写在 React 组件中：

```text
character-pose/
├── utils/
│   ├── mirrorSkeletonPose.ts
│   ├── skeletonSide.ts
│   └── skeletonSerialization.ts
├── state/
│   └── poseStore.ts
└── components/
    └── PosePresetPanel.tsx
```

核心职责：

- `skeletonSide.ts`：判断 bone 左右侧、计算对侧 bone name。
- `mirrorSkeletonPose.ts`：输入 skeleton pose，输出镜像后的 skeleton patch。
- `poseStore.ts`：提供 `mirrorCharacterPose(characterId)` action。
- `PosePresetPanel.tsx`：只负责渲染按钮并调用 action。

### 6.2 推荐类型

```ts
export interface PoseRotation {
  x: number
  y: number
  z: number
}

export interface PoseVector3 {
  x: number
  y: number
  z: number
}

export interface PoseQuaternion {
  x: number
  y: number
  z: number
  w: number
}

export interface CharacterPoseBone {
  name: string
  rotation?: PoseRotation
  position?: PoseVector3
  quaternion?: PoseQuaternion
}

export type CharacterSkeletonPose = Record<string, CharacterPoseBone>
```

### 6.3 推荐纯函数实现

```ts
import * as THREE from 'three'

export function getOppositeBoneName(name: string): string {
  if (name.includes('Left')) return name.replace('Left', 'Right')
  if (name.includes('Right')) return name.replace('Right', 'Left')
  return name
}

export function mirrorRotation(rotation: PoseRotation): PoseRotation {
  const quat = new THREE.Quaternion().setFromEuler(
    new THREE.Euler(rotation.x, rotation.y, rotation.z)
  )

  quat.x *= -1
  quat.w *= -1

  const euler = new THREE.Euler().setFromQuaternion(quat)
  return { x: euler.x, y: euler.y, z: euler.z }
}

export function mirrorSkeletonPose(
  skeleton: CharacterSkeletonPose
): CharacterSkeletonPose {
  const mirrored: CharacterSkeletonPose = {}

  for (const [sourceName, sourceBone] of Object.entries(skeleton)) {
    if (!sourceBone.rotation) continue

    const targetName = getOppositeBoneName(sourceName)

    mirrored[targetName] = {
      ...sourceBone,
      name: targetName,
      rotation: mirrorRotation(sourceBone.rotation),
      // position / quaternion 建议作为运行期字段，不作为镜像结果的强依赖。
      position: sourceBone.position
        ? { ...sourceBone.position }
        : undefined,
      quaternion: undefined
    }
  }

  return mirrored
}
```

如果 FlowCanvas 的姿势更新接口沿用数组 patch，可在 action 层转换：

```ts
const patch = Object.values(mirrorSkeletonPose(character.skeleton))
updateCharacterIkSkeleton({ characterId, skeleton: patch })
```

### 6.4 推荐状态 action

```ts
mirrorCharacterPose(characterId: string) {
  const character = getCharacter(characterId)
  if (!character?.skeleton) return

  const mirrored = mirrorSkeletonPose(character.skeleton)
  updateCharacterSkeletonPatch(characterId, Object.values(mirrored))
}
```

UI 组件只调用：

```tsx
<button onClick={() => mirrorCharacterPose(character.id)}>
  镜像姿势
</button>
```

## 7. 需要改进的边界情况

Storyboarder 当前实现较直接，FlowCanvas 移植时建议补强以下情况。

### 7.1 空 skeleton

当前实现直接：

```js
let keys = Object.keys(originalSkeleton)
```

如果 `sceneObject.skeleton` 为空或不存在，会报错。FlowCanvas 应保护：

```ts
if (!skeleton || Object.keys(skeleton).length === 0) return {}
```

### 7.2 缺少 rotation

当前实现默认每个 bone 都有 `rotation`：

```js
let boneRot = originalSkeleton[key].rotation
```

FlowCanvas 应跳过没有 rotation 的 bone，或用默认 `{x:0,y:0,z:0}`。

### 7.3 缺少 position

当前实现默认每个 bone 都有 `position`：

```js
position: { x: position.x, y: position.y, z: position.z }
```

但序列化保存时会移除 `position`，所以 FlowCanvas 不能假设它一定存在。

建议：

- 镜像结果只强制写 `name` 和 `rotation`。
- `position` 和 `quaternion` 由角色重新渲染后的补全逻辑生成。

### 7.4 `id` 字段不稳定

当前实现写：

```js
id: originalSkeleton[boneName].id
```

但骨骼状态中不总是有 `id`。`UPDATE_CHARACTER_IK_SKELETON` reducer 也不依赖 `id`，只依赖 `bone.name`。

FlowCanvas 建议不要依赖 skeleton bone patch 中的 `id`，除非有单独的 bone selection 系统需要它。

### 7.5 左右命名规则

Storyboarder 只处理包含 `Left` / `Right` 的 bone。FlowCanvas 如果要支持更多模型，建议支持常见命名：

```text
Left / Right
L_ / R_
.L / .R
_L / _R
left / right
```

但内部仍应统一映射到标准骨骼名，避免 IK 逻辑复杂化。

## 8. 验收标准

### 基础行为

- 选中人物后，姿势页签中显示“镜像姿势”按钮。
- 点击按钮后，左侧肢体姿势与右侧肢体姿势互换。
- 中轴骨骼获得镜像后的旋转。
- 镜像后角色立即刷新，不需要重新选择角色。

### 状态行为

- 点击镜像后，角色 `skeleton` 被更新。
- 更新通过统一 pose action 完成，而不是直接操作 Three.js bone。
- 镜像后可继续使用 IK 控制点和 bone 旋转编辑。
- 镜像操作可被 undo / redo 系统记录。

### 持久化

- 镜像后保存项目，再重新打开，姿势保持一致。
- 保存数据只依赖稳定字段：`name` 和 `rotation`。
- 不依赖运行期 `position`、`quaternion` 或 Three.js object uuid。

### 边界情况

- 空 skeleton 不报错。
- 只有单侧骨骼时不报错。
- 缺少 `position` / `quaternion` 时不报错。
- 自定义模型命名不标准时，给出明确降级或提示。

## 9. 最小移植步骤

1. **抽出纯函数**
   - 实现 `getOppositeBoneName()`。
   - 实现 `mirrorRotation()`。
   - 实现 `mirrorSkeletonPose()`。

2. **接入状态**
   - 在 FlowCanvas pose store 中新增 `mirrorCharacterPose(characterId)`。
   - 内部读取当前角色 skeleton，生成 mirrored skeleton patch 并写回。

3. **接入 UI**
   - 在人物属性 / 姿势预设面板中增加“镜像姿势”按钮。
   - 按钮只触发状态 action。

4. **接入 undo / redo**
   - 镜像前开启 undo group。
   - 写回 skeleton patch。
   - 镜像后关闭 undo group。

5. **补充测试**
   - 单测 `mirrorRotation()` 和 `getOppositeBoneName()`。
   - 单测左右骨骼交换。
   - 手动验证标准人物模型的镜像效果。

## 10. 与完整姿势编辑移植的关系

镜像姿势不依赖 IK 控制器本身，但依赖同一套 skeleton 数据结构和姿势应用流程。

因此 FlowCanvas 中建议按以下顺序实现：

```text
角色 skeleton 应用
-> 单 bone 姿势更新
-> 镜像姿势
-> IK 控制点编辑
-> 姿势预设保存 / 应用
```

如果 FlowCanvas 已经完成 `sceneObject.skeleton -> CharacterModel bone rotation` 的应用链路，那么镜像姿势只需要新增一个纯数据转换 action，复杂度远低于完整 IK 移植。
