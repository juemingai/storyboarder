# Shot Generator 人物姿势编辑与 IK 功能移植分析

本文档说明 Storyboarder Shot Generator 中“人物模型调整姿势 / 编辑动作”的实现方式、核心代码位置和移植到 FlowCanvas 时的建议拆分方案。

该功能本质上由四层组成：

- 人物 GLB / SkinnedMesh 加载与骨骼姿势应用
- 骨骼直接旋转编辑
- IK 控制点拖拽编辑
- 姿势状态写回、预设保存与序列化

## 1. 功能总览

Shot Generator 中人物姿势编辑支持两类操作：

1. **直接旋转骨骼**
   - 点击人物骨骼后显示旋转控制器。
   - 用户旋转单根 bone。
   - 写回 `sceneObject.skeleton[boneName].rotation`。

2. **IK 控制点拖拽**
   - 选中人物后，在头、手、脚、髋部显示 IK 控制点。
   - 用户拖动控制点，IK 系统求解整条肢体链。
   - 解算结果批量写回人物骨骼。

人物姿势最终存储在 character 对象的 `skeleton` 字段中。角色重新渲染时，`Character` 组件读取该字段并重新应用到 Three.js bone。

## 2. 核心文件位置

### 2.1 主场景入口

- `src/js/shot-generator/SceneManagerR3fLarge.js`

职责：

- 创建并初始化 `SGIkHelper`。
- 将 Redux action 包装成 IK helper 可调用的回调。
- 渲染所有 character，并把 `updateCharacterSkeleton`、`updateCharacterIkSkeleton` 传给 `Character`。

关键逻辑：

```js
let sgIkHelper = SGIkHelper.getInstance()
sgIkHelper.setUp(null, rootRef.current, camera, gl.domElement)

const updateSkeleton = skeleton => {
  updateCharacterIkSkeleton({
    id: sgIkHelper.characterObject.userData.id,
    skeleton
  })
}

sgIkHelper.setUpdate(
  updateCharacterRotation,
  updateSkeleton,
  updateCharacterPos,
  updatePoleTarget,
  updateObjects
)
```

### 2.2 人物渲染与姿势应用

- `src/js/shot-generator/components/Three/Character.js`

职责：

- 加载人物模型。
- 提取 `SkinnedMesh.skeleton`。
- 将 `sceneObject.skeleton` 中的姿势应用到 Three.js bone。
- 角色选中时挂载骨骼 helper、IK helper 和旋转控制器。
- 在姿势变化后调用 `fullyUpdateIkSkeleton()` 补全骨骼位置、旋转和 quaternion。

关键实现点：

- 使用 `cloneGltf(gltf)` 克隆模型，避免多个角色共享 skeleton。
- 从模型中查找 `SkinnedMesh`，构造 `LOD`。
- 调用 `skeleton.pose()` 得到初始姿势。
- 克隆一份 `originalSkeleton`，作为未修改 bone 的 fallback。

姿势应用逻辑：

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

角色选中时挂载 IK：

```js
if (isIkCharacter.current && !SGIkHelper.getInstance().isIkDisabled) {
  SGIkHelper.getInstance().initialize(ref.current, originalHeight, lod.children[0], sceneObject)
  ref.current.add(SGIkHelper.getInstance())
}
```

批量补全骨骼状态：

```js
changedSkeleton.push({
  name: bone.name,
  position: { x: position.x, y: position.y, z: position.z },
  rotation: { x: rotation.x, y: rotation.y, z: rotation.z },
  quaternion: { x: quaternion.x, y: quaternion.y, z: quaternion.z, w: quaternion.w }
})
```

### 2.3 用户交互入口

- `src/js/shot-generator/components/Three/InteractionManager.js`

职责：

- 处理主 3D Canvas 的 pointer 事件。
- 优先检测 IK 控制点命中。
- 点击控制点时调用 `SGIkHelper.selectControlPoint()`。
- pointer up 时调用 `SGIkHelper.deselectControlPoint()`，触发姿势写回。

命中检测优先级：

```js
let intersects = raycaster.current.intersectObject(SGIkHelper.getInstance())
if (intersects.length > 0) {
  return intersects
}
```

点击 IK 控制点：

```js
if (target.userData.type === 'controlPoint' || target.userData.type === 'poleTarget') {
  SGIkHelper.getInstance().selectControlPoint(target.uuid, event)
  target = characters[0]
  isSelectedControlPoint = true
}
```

释放鼠标：

```js
SGIkHelper.getInstance().deselectControlPoint(event)
```

拖动过程中，如果 IK 正在工作，则跳过普通物体拖拽：

```js
let ikRig = SGIkHelper.getInstance().ragDoll
if (!ikRig || !ikRig.isEnabledIk && !ikRig.hipsMoving && !ikRig.hipsMouseDown) {
  drag({ x, y }, dragTarget.target, camera, selections, event.ctrlKey)
}
```

### 2.4 IK Helper

- `src/js/shared/IK/SGIkHelper.js`

职责：

- 作为全局单例管理当前选中角色的 IK 控制系统。
- 创建可交互控制点。
- 创建 pole target。
- 初始化 `Ragdoll` IK rig。
- 在控制点选择、拖动、释放时协调 IK 状态。

创建的控制点：

```text
Head
LeftHand
RightHand
LeftLeg
RightLeg
Hips
```

创建的 pole target：

```text
leftArmPole
rightArmPole
leftLegPole
rightLegPole
```

初始化流程：

```js
setUp(mesh, scene, camera, domElement)
  -> 创建 controlPoints / poleTargets / transformControls
  -> initializeInstancedMesh()
  -> 创建 ControlTargetSelection
  -> 创建 Ragdoll

initialize(object, height, skinnedMesh, props)
  -> initPoleTarget()
  -> ragDoll.cleanUp()
  -> ragDoll.initObject()
  -> ragDoll.reinitialize()
  -> updateAllTargetPoints()
```

控制点释放时会写回姿势：

```js
deselectControlPoint() {
  this.ragDoll.updateReact()
  this.updateObjects(changes)
}
```

### 2.5 IK 解算核心

核心文件：

- `src/js/shared/IK/objects/IkObjects/IkObject.js`
- `src/js/shared/IK/objects/IkObjects/Ragdoll.js`
- `src/js/shared/IK/objects/IkObjects/IkSwitcher.js`
- `src/js/shared/IK/core/three-ik.js`

#### IkObject

`IkObject` 是 IK rig 的基类。

职责：

- 克隆角色骨架。
- 创建 IK chain。
- 将控制点绑定到 chain effector。
- 调用 `three-ik` 求解。

定义的 chain：

```js
this.chainObjects["Head"] = new ChainObject("Spine", "Head", controlTargets[1])
this.chainObjects["LeftHand"] = new ChainObject("LeftArm", "LeftHand", controlTargets[2])
this.chainObjects["RightHand"] = new ChainObject("RightArm", "RightHand", controlTargets[3])
this.chainObjects["LeftFoot"] = new ChainObject("LeftUpLeg", "LeftFoot", controlTargets[4])
this.chainObjects["RightFoot"] = new ChainObject("RightUpLeg", "RightFoot", controlTargets[5])
```

IK 求解入口：

```js
if (this.isEnabledIk) {
  if (!this.isRotation) {
    let target = this.getTargetForSolve()
    this.ik.solve(target)
  }
  this.lateUpdate()
}
```

#### Ragdoll

`Ragdoll` 继承 `IkObject`，负责 Shot Generator 人形角色的具体 IK 行为。

职责：

- 创建 pole target。
- 维护髋部、背部、头部和四肢的控制逻辑。
- IK 解算后把结果同步回原始角色。
- `updateReact()` 将变化后的 bone 转成可写入状态的数据。

姿势写回：

```js
for (let bone of skinnedMesh.skeleton.bones) {
  if (!this.ikSwitcher.ikBonesName.some(boneName => bone.name === boneName)) {
    continue
  }

  changedSkeleton.push({
    name: bone.name,
    position: { x: position.x, y: position.y, z: position.z },
    rotation: { x: rotation.x, y: rotation.y, z: rotation.z }
  })
}

this.updateCharacterSkeleton(changedSkeleton)
```

#### IkSwitcher

`IkSwitcher` 负责在“克隆 IK 骨架”和“原始角色骨架”之间转换旋转。

存在它的原因：

- Storyboarder 的角色模型默认朝向与 `three-ik` 的计算坐标系不完全一致。
- IK 解算发生在克隆骨架上。
- 解算结果需要通过相对 quaternion 转换后应用回原始角色骨架。

关键方法：

- `recalculateDifference()`
- `calculateRelativeAngle()`
- `applyChangesToOriginal()`
- `applyToIk()`

## 3. 状态管理与数据结构

### 3.1 Character 对象中的姿势字段

新建人物时，默认姿势来自 pose preset：

- `src/js/shared/actions/scene-object-creators.js`

```js
return createObject({
  id,
  type: 'character',
  height: 1.8,
  model: 'adult-male',
  posePresetId: defaultPosePreset.id,
  skeleton: defaultPosePreset.state.skeleton,
  visible: true
})
```

典型 `skeleton` 结构：

```js
{
  Hips: {
    name: 'Hips',
    rotation: { x, y, z },
    position: { x, y, z },
    quaternion: { x, y, z, w }
  },
  LeftArm: {
    name: 'LeftArm',
    rotation: { x, y, z }
  }
}
```

其中：

- `rotation` 是最终姿势恢复的核心字段。
- `position` 和 `quaternion` 主要用于 IK 运行期计算。
- 保存 storyboarder 文件时会省略 `position` 和 `quaternion`。

### 3.2 Reducer

- `src/js/shared/reducers/shot-generator.js`

单根骨骼更新：

```js
case 'UPDATE_CHARACTER_SKELETON':
  draft[id].skeleton[name].rotation = { x, y, z }
```

IK 批量骨骼更新：

```js
case 'UPDATE_CHARACTER_IK_SKELETON':
  for (let bone of action.payload.skeleton) {
    draft[id].skeleton[bone.name].rotation = { x, y, z }
    draft[id].skeleton[bone.name].position = { x, y, z }
    draft[id].skeleton[bone.name].quaternion = { x, y, z, w }
  }
```

Pole target 更新：

```js
case 'UPDATE_CHARACTER_IK_POLETARGETS':
  draft[id].poleTargets[key] = value
```

Action creator：

```js
updateCharacterSkeleton: ({ id, name, rotation }) => ({
  type: 'UPDATE_CHARACTER_SKELETON',
  payload: { id, name, rotation }
})

updateCharacterIkSkeleton: ({ id, skeleton }) => ({
  type: 'UPDATE_CHARACTER_IK_SKELETON',
  payload: { id, skeleton }
})

updateCharacterPoleTargets: ({ id, poleTargets }) => ({
  type: 'UPDATE_CHARACTER_IK_POLETARGETS',
  payload: { id, poleTargets }
})
```

### 3.3 序列化

- `src/js/shared/reducers/shot-generator/serialize-scene-object.js`

保存角色时，会去掉运行期字段：

```js
skeleton: when(isPresent,
  map(
    omit(['quaternion', 'position'])
  )
)
```

这说明 FlowCanvas 持久化时也可以只保存 `rotation`，把 `position` 和 `quaternion` 作为运行期缓存。

## 4. 骨骼命名与模型适配

- `src/js/utils/isSuitableForIk.js`

Shot Generator 的 IK 系统依赖标准人形骨骼命名：

```text
Hips
Spine
Spine1
Spine2
Neck
Head
LeftShoulder
LeftArm
LeftForeArm
LeftHand
RightShoulder
RightArm
RightForeArm
RightHand
LeftUpLeg
LeftLeg
LeftFoot
RightUpLeg
RightLeg
RightFoot
```

实现中使用 `bone.name.includes(name)` 做宽松匹配，并会把匹配到的 bone 重命名为标准名称：

```js
bone.name = ikBoneName
bone.userData.name = ikBoneName
```

移植到 FlowCanvas 时应保留一个显式的模型适配步骤：

- 加载 GLB 后扫描 skeleton。
- 检查是否包含全部 IK 必需骨骼。
- 建立原始 bone name 到标准 bone name 的映射。
- 不建议静默修改用户模型，最好在导入阶段提示“该模型是否支持姿势编辑”。

## 5. 姿势预设与检查器

### 5.1 姿势预设

相关文件：

- `src/js/shot-generator/components/PosePresetsEditor/index.js`
- `src/js/shot-generator/components/PosePresetsEditor/PosePresetEditorItem.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js`

职责：

- 展示系统或用户保存的姿势预设。
- 点击预设后更新角色的 `posePresetId` 和 `skeleton`。
- 保存当前角色 skeleton 为新预设。
- 支持镜像 / 反向姿势一类操作。

应用姿势预设：

```js
let posePresetId = preset.id
let skeleton = preset.state.skeleton
updateObject(id, { posePresetId, skeleton })
```

### 5.2 骨骼检查器

- `src/js/shot-generator/components/InspectedElement/BoneInspector/index.js`

职责：

- 显示当前选中 bone 名称。
- 提供 x/y/z 三个角度 slider。
- slider 修改后调用 `updateCharacterSkeleton()`。

该检查器只处理“单根骨骼旋转”，不参与 IK 求解。

## 6. 运行时数据流

### 6.1 选中人物

```text
用户点击人物
-> InteractionManager 选中 character id
-> Character 收到 isSelected=true
-> 初始化 BonesHelper
-> 如果 skeleton 适合 IK，初始化 SGIkHelper
-> 将 SGIkHelper 挂到 character group 上
```

### 6.2 直接旋转骨骼

```text
用户点击骨骼
-> InteractionManager 命中 BonesHelper
-> selectBone(bone.uuid)
-> Character 用 ObjectRotationControl 选中该 bone
-> 用户旋转 bone
-> ObjectRotationControl transformMouseUp
-> updateCharacterSkeleton({ id, name, rotation })
-> reducer 更新 sceneObject.skeleton
-> Character 根据 skeleton 重新应用姿势
```

### 6.3 IK 拖动手脚头髋

```text
用户点击 IK 控制点
-> InteractionManager 优先命中 SGIkHelper
-> SGIkHelper.selectControlPoint()
-> Ragdoll.isEnabledIk = true
-> TransformControls 接管控制点
-> 每帧 SGIkHelper.update()
-> Ragdoll.update()
-> three-ik solve(target)
-> IkSwitcher.applyChangesToOriginal()
-> 用户释放鼠标
-> SGIkHelper.deselectControlPoint()
-> Ragdoll.updateReact()
-> updateCharacterIkSkeleton({ id, skeleton })
-> reducer 批量更新 sceneObject.skeleton
-> Character 重新应用姿势
```

### 6.4 Pole target 调整

```text
用户拖动 pole target
-> SGIkHelper.deselectControlPoint()
-> 计算 pole target 在角色局部坐标中的位置
-> updateCharacterPoleTargets({ id, poleTargets })
-> sceneObject.poleTargets 更新
-> Character 监听 poleTargets 变化
-> SGIkHelper.initPoleTarget(originalHeight, sceneObject, true)
```

## 7. FlowCanvas 移植建议

### 7.1 不建议直接照搬 Redux 结构

Storyboarder 的实现深度绑定 Redux、旧版 R3F 和旧版 Three.js。FlowCanvas 应抽象为独立姿势模块：

```text
character-pose/
├── components/
│   ├── CharacterModel.tsx
│   ├── CharacterPoseControls.tsx
│   └── BoneRotationInspector.tsx
├── controllers/
│   ├── CharacterPoseController.ts
│   ├── IkPoseController.ts
│   └── BoneRotationController.ts
├── ik/
│   ├── SGIkHelper.ts
│   ├── Ragdoll.ts
│   ├── IkObject.ts
│   ├── IkSwitcher.ts
│   └── threeIkAdapter.ts
├── state/
│   └── poseStore.ts
├── types/
│   └── poseTypes.ts
└── utils/
    ├── skeletonMapping.ts
    ├── skeletonSerialization.ts
    └── characterCoordinate.ts
```

如果 FlowCanvas 已有 3D 导演台目录，可以把这些模块合并到现有结构中。

### 7.2 推荐类型设计

```ts
export interface CharacterPoseBone {
  name: string
  rotation?: { x: number; y: number; z: number }
  position?: { x: number; y: number; z: number }
  quaternion?: { x: number; y: number; z: number; w: number }
}

export type CharacterSkeletonPose = Record<string, CharacterPoseBone>

export interface CharacterPoleTarget {
  position: { x: number; y: number; z: number }
  currentCharacterHeight?: number
}

export interface CharacterObject {
  id: string
  type: 'character'
  model: string
  height: number
  x: number
  y: number
  z: number
  rotation: number
  skeleton: CharacterSkeletonPose
  poleTargets?: Record<string, CharacterPoleTarget>
  posePresetId?: string | null
}
```

### 7.3 推荐状态接口

```ts
interface PoseActions {
  updateCharacterBoneRotation(input: {
    characterId: string
    boneName: string
    rotation: { x: number; y: number; z: number }
  }): void

  updateCharacterIkSkeleton(input: {
    characterId: string
    skeleton: CharacterPoseBone[]
  }): void

  updateCharacterPoleTargets(input: {
    characterId: string
    poleTargets: Record<string, CharacterPoleTarget>
  }): void
}
```

### 7.4 坐标系注意事项

Storyboarder 渲染角色时做了坐标映射：

```jsx
position={[x, z, y]}
rotation={[0, rotation, 0]}
scale={[bodyScale, bodyScale, bodyScale]}
```

也就是说，状态中的 `y` 被映射到 Three.js 的 `z`，状态中的 `z` 被映射到 Three.js 的 `y`。

FlowCanvas 移植时必须先确认自身坐标系：

- 如果 FlowCanvas 直接使用 Three.js 标准坐标：`Y` 为上、`XZ` 为地面，则不要照搬 `[x, z, y]`。
- 如果 FlowCanvas 沿用 Storyboarder 导演台状态模型，则保留映射函数，并集中放在 `characterCoordinate.ts` 中。

### 7.5 Three.js 版本风险

当前项目依赖：

- `three@^0.115.0`
- `react-three-fiber@4.0.12`

现代 FlowCanvas 如果使用较新版本 Three.js / R3F，需要适配：

- `Matrix4.getInverse(matrix)` 已被新版 Three.js 移除，应改为 `matrix.copy(source).invert()`。
- `Geometry` / `BufferGeometry` API 差异。
- `react-three-fiber` v4 到 `@react-three/fiber` 的 hook 和事件系统差异。
- `TransformControls` 的事件名和挂载方式可能变化。
- `THREE.Math` 已迁移为 `THREE.MathUtils`。

建议先将 IK 代码封装为独立 adapter，避免散落在 React 组件中。

## 8. 最小移植路径

建议按以下顺序迁移，降低风险：

1. **角色加载与 skeleton 应用**
   - 先实现 `CharacterModel`。
   - 支持从状态读取 `skeleton` 并应用 bone rotation。
   - 暂不做 IK。

2. **骨骼选择与单 bone 旋转**
   - 接入 `BonesHelper` 或自研骨骼拾取。
   - 接入 `TransformControls`。
   - 实现 `updateCharacterBoneRotation()`。

3. **IK 控制点显示**
   - 移植 `SGIkHelper` 的控制点创建逻辑。
   - 先只显示控制点和 pole target，不求解。

4. **IK 解算**
   - 移植 `three-ik`、`IkObject`、`Ragdoll`、`IkSwitcher`。
   - 完成手、脚、头、髋部拖动。

5. **姿势预设**
   - 保存 / 应用 `skeleton`。
   - 序列化时只保存必要字段。

6. **模型适配**
   - 增加 skeleton 验证。
   - 对不支持 IK 的模型降级为“只允许直接旋转骨骼”或禁用姿势编辑。

## 9. 移植验收标准

### 基础姿势

- 加载标准人物模型后，默认姿势正确。
- 应用已有 `skeleton` 后，人物姿势恢复一致。
- 切换姿势预设后，角色姿势立即更新。

### 骨骼旋转

- 点击 bone 后能显示旋转控制器。
- 旋转 bone 后，状态中对应 bone 的 `rotation` 更新。
- 取消选择再选中，姿势不丢失。

### IK 控制

- 选中人物后显示头、手、脚、髋部控制点。
- 拖动左右手时，手臂链自然跟随。
- 拖动左右脚时，腿部链自然跟随。
- 拖动头部时，脊柱和头部姿势更新。
- 拖动髋部时，人物整体位置或髋部姿势符合 FlowCanvas 的交互定义。
- 调整 pole target 后，肘部 / 膝盖朝向稳定。

### 持久化

- 保存后重新打开，人物姿势一致。
- 保存数据不依赖运行期 Three.js 对象。
- `position` / `quaternion` 如仅用于运行期，可不持久化。

## 10. 关键依赖清单

移植时最少需要关注以下代码：

```text
src/js/shot-generator/components/Three/Character.js
src/js/shot-generator/components/Three/InteractionManager.js
src/js/shot-generator/SceneManagerR3fLarge.js
src/js/shared/reducers/shot-generator.js
src/js/shared/reducers/shot-generator/serialize-scene-object.js
src/js/shared/actions/scene-object-creators.js
src/js/shared/IK/SGIkHelper.js
src/js/shared/IK/objects/IkObjects/IkObject.js
src/js/shared/IK/objects/IkObjects/Ragdoll.js
src/js/shared/IK/objects/IkObjects/IkSwitcher.js
src/js/shared/IK/objects/TargetControl.js
src/js/shared/IK/objects/ObjectRotationControl.js
src/js/shared/IK/core/three-ik.js
src/js/shared/IK/utils/SkeletonUtils.js
src/js/shared/IK/utils/TransformControls.js
src/js/shared/IK/utils/Object3dExtension.js
src/js/utils/isSuitableForIk.js
```

如果只做第一阶段“姿势恢复 + 单骨骼旋转”，可以先不移植完整 `src/js/shared/IK`，只保留：

```text
Character.js 中的 skeleton 应用逻辑
ObjectRotationControl.js
BoneInspector/index.js
shot-generator.js 中的 UPDATE_CHARACTER_SKELETON
```

如果要实现 Shot Generator 同级别的“拖动手脚自动摆姿势”，则必须移植完整 IK 子系统。
