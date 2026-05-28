# Shot Generator 人物配件绑定实现分析

本文分析 Shot Generator 在人物编辑状态下，点击截图中的“配件/道具”（代码里称为 `attachable`）后，如何创建一个配件对象并把它绑定到人物骨骼上。

## 结论

Shot Generator 的“配件/道具绑定人物”不是通过物理约束或 Three.js `SkinnedMesh.bind()` 实现的，而是通过以下机制完成：

1. 在 Redux `sceneObjects` 中创建一个 `type: 'attachable'` 的场景对象。
2. 该对象用 `attachToId` 记录被绑定的人物 id，用 `bindBone` 记录绑定到哪根骨骼。
3. 渲染时 `Attachable` 组件找到目标人物的 `SkinnedMesh.skeleton`，再通过 `skeleton.getBoneByName(bindBone)` 找到骨骼。
4. 调用 `bone.add(ref.current)`，把配件的 Three.js `Group` 直接挂到该骨骼节点下面。
5. 配件的位置、旋转在 Redux 中以世界坐标形式保存；每次写入/读取时通过父骨骼矩阵和逆矩阵在世界空间与骨骼局部空间之间转换。

核心代码路径：

- 创建配件：`src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js`
- 渲染与绑定：`src/js/shot-generator/components/Three/Attachable.js`
- 场景中挂载配件组件：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 点击选择与拖拽：`src/js/shot-generator/components/Three/InteractionManager.js`
- 拖拽坐标转换：`src/js/shot-generator/hooks/use-dragging-manager.js`
- Redux actions/reducers：`src/js/shared/reducers/shot-generator.js`

## 数据结构

点击一个配件模型后，代码会创建类似下面的 `sceneObject`：

```js
{
  id: key,
  type: 'attachable',
  x: modelData.x,
  y: modelData.y,
  z: modelData.z,
  model: modelData.id,
  name: modelData.name,
  bindBone: name || modelData.bindBone,
  attachToId: id,
  size: 1,
  status: 'PENDING',
  rotation: modelData.rotation
}
```

字段含义：

- `type: 'attachable'`：标记这是配件/可附着对象。
- `attachToId`：被绑定人物的 `sceneObject.id`。
- `bindBone`：绑定的骨骼名称，例如 `Head`、手部骨骼等。
- `model`：配件模型 id 或用户导入模型路径。
- `x/y/z`：配件世界空间位置。
- `rotation`：配件世界空间旋转。
- `size`：配件缩放系数。
- `status: 'PENDING'`：首次创建后需要在 Three 场景中初始化定位。

参考：`src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js:85`。

## 创建流程

### 1. 点击配件模型

人物 Inspector 中的配件页签使用 `AttachableInspector`。用户点击某个配件卡片时，触发 `onSelectItem(model)`：

- 如果模型自带 `bindBone`，且当前人物不是用户自定义模型，则直接创建配件。
- 否则打开 `HandSelectionModal`，让用户选择绑定骨骼。

关键代码：`src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js:74`。

```js
const onSelectItem = useCallback((model) => {
  selectedModel.current = model
  selectedId.current = model.id || id
  if(model.bindBone && !isUserModel(sceneObject.model)) {
    createAttachableElement(model, id)
    return
  }

  showModal(true)
}, [id, sceneObject])
```

### 2. 创建 Redux 场景对象

`createAttachableElement` 生成 uuid，组装 `type: 'attachable'` 对象，然后依次：

1. `undoGroupStart()` 开启 undo 分组。
2. `createObject(element)` 写入 `sceneObjects`。
3. `selectAttachable({ id: element.id, bindId: element.attachToId })` 选中配件，同时保持人物为当前选择对象。
4. `undoGroupEnd()` 结束 undo 分组。

参考：`src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js:108`。

Redux 中 `CREATE_OBJECT` 只是把对象插入 `sceneObjects`：`src/js/shared/reducers/shot-generator.js:937`。

`SELECT_ATTACHABLE` 有两个效果：

- `selections` 被设置为 `[bindId]`，也就是仍然选中人物。
- `selectedAttachable` 被设置为配件 id，用于让配件显示选中态和旋转控制。

参考：`src/js/shared/reducers/shot-generator.js:828`、`src/js/shared/reducers/shot-generator.js:864`。

## 渲染与绑定流程

### 1. SceneManager 渲染所有 attachable

`SceneManagerR3fLarge` 会从 `sceneObjects` 中筛出 `type === 'attachable'` 的 id，然后为每个配件渲染 `Attachable`：

```jsx
<Attachable
  path={ModelLoader.getFilepathForModel(sceneObject, {storyboarderFilePath})}
  sceneObject={sceneObject}
  isSelected={selectedAttachable === sceneObject.id}
  updateObject={updateObject}
  сharacterModelPath={ModelLoader.getFilepathForModel(sceneObjects[sceneObject.attachToId], {storyboarderFilePath})}
  deleteObjects={deleteObjects}
  character={sceneObjects[sceneObject.attachToId]}
  withState={withState}
/>
```

这里的重点是：

- `sceneObject` 是配件自身。
- `character` 是 `sceneObjects[attachToId]`，也就是目标人物。
- `сharacterModelPath` 用来加载同一个人物模型，确保配件绑定时能拿到骨架信息。

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:423`。

### 2. Attachable 加载模型并构造 Three Group

`Attachable` 内部通过 `useAsset(path)` 加载配件 glTF，通过 `useAsset(сharacterModelPath)` 加载人物模型。配件 glTF 中每个 Mesh 会被 clone 出来并放进一个 React Three Fiber `<group ref={ref}>` 内。

该 group 的 `userData`：

```js
userData={{
  type: 'attachable',
  id: sceneObject.id,
  bindedId: sceneObject.attachToId,
  isRotationEnabled: false
}}
```

`type` 用于点击/拖拽判断，`bindedId` 用于反查它绑定的人物。

参考：`src/js/shot-generator/components/Three/Attachable.js:379`。

### 3. 首次初始化时挂到目标骨骼

`Attachable` 初始化时会：

1. 从 Three 场景中找到目标人物对象：`userData.id === sceneObject.attachToId`。
2. 找到人物里的 `SkinnedMesh`。
3. 通过 `skinnedMesh.skeleton.bones.find(...)` 或 `skeleton.getBoneByName(sceneObject.bindBone)` 找到绑定骨骼。
4. 如果 `status === 'PENDING'`，先计算初始世界位置/旋转并写回 Redux，随后把状态改为 `Loaded`。
5. 调用 `bone.add(ref.current)`，把配件 Group 变成骨骼的子节点。

关键绑定代码：`src/js/shot-generator/components/Three/Attachable.js:140`。

```js
let skinnedMesh = characterObject.current.getObjectByProperty('type', 'SkinnedMesh')
let bone = skinnedMesh.skeleton.bones.find(b => b.name === sceneObject.bindBone)

if(sceneObject.status === 'PENDING') {
  // 计算初始世界位置/旋转，并 updateObject(..., { status: 'Loaded' })
}

bone.add(ref.current)
```

这就是“道具跟着人物身体/骨骼动”的核心：Three.js 的层级变换会让骨骼子节点继承骨骼的矩阵，所以人物换 pose、移动骨骼时，配件自动跟随。

## 坐标保存方式

虽然渲染时配件挂在骨骼下面，但 Redux 里保存的是接近世界空间的 `x/y/z/rotation`。因此代码里经常做矩阵转换。

### 1. Redux 位置变化时：世界坐标转骨骼局部坐标

当 `sceneObject.x/y/z` 变化时，`Attachable` 先把当前配件应用到父骨骼世界矩阵，再设置世界位置，最后应用父骨骼逆矩阵转回局部空间：

参考：`src/js/shot-generator/components/Three/Attachable.js:245`。

```js
let parentMatrixWorld = ref.current.parent.matrixWorld
let parentInverseMatrixWorld = ref.current.parent.getInverseMatrixWorld()
ref.current.applyMatrix4(parentMatrixWorld)
ref.current.position.set(sceneObject.x, sceneObject.y, sceneObject.z)
ref.current.updateMatrixWorld(true)
ref.current.applyMatrix4(parentInverseMatrixWorld)
ref.current.updateMatrixWorld(true)
```

旋转变化时也使用同一思路：先转到世界空间，设置世界旋转，再转回父骨骼局部空间。参考：`src/js/shot-generator/components/Three/Attachable.js:258`。

### 2. 保存回 Redux 时：骨骼局部坐标转世界坐标

`saveToStore()` 会把配件局部矩阵乘上父骨骼 `matrixWorld`，再 `decompose` 得到世界位置和世界旋转，写回 Redux。

参考：`src/js/shot-generator/components/Three/Attachable.js:330`。

```js
let position = ref.current.worldPosition()
let quaternion = ref.current.worldQuaternion()
let matrix = ref.current.matrix.clone()
matrix.premultiply(ref.current.parent.matrixWorld)
matrix.decompose(position, quaternion, new THREE.Vector3())
let rot = new THREE.Euler().setFromQuaternion(quaternion, 'XYZ')

updateObject(sceneObject.id, {
  x: position.x,
  y: position.y,
  z: position.z,
  rotation: { x: rot.x, y: rot.y, z: rot.z }
})
```

## 换骨骼/重新绑定

配件卡片上可以修改 “Attached to bone”。修改后调用：

```js
updateObject(id, { bindBone: name })
```

参考：`src/js/shot-generator/components/InspectedElement/AttachableEditor/index.js:82`。

`Attachable` 监听 `sceneObject.bindBone`，一旦变化就执行 `rebindAttachable()`：

1. 重新找到目标人物。
2. 重新找到 `sceneObject.bindBone` 对应骨骼。
3. `bone.add(ref.current)` 把配件换挂到新骨骼。
4. 通过 offset 差值尽量保持视觉位置不跳。
5. 重新计算缩放并 `saveToStore()`。

参考：`src/js/shot-generator/components/Three/Attachable.js:270`。

```js
let skeleton = skinnedMesh.skeleton
let bone = skeleton.getBoneByName(sceneObject.bindBone)
bone.add(ref.current)

let currentOffset = getOffsetBetweenCharacterAndAttachable()
let offsetDifference = currentOffset.sub(offsetToCharacter.current)
ref.current.parent.localToWorld(ref.current.position)
ref.current.position.add(offsetDifference)
ref.current.parent.worldToLocal(ref.current.position)

saveToStore()
```

## 缩放逻辑

配件缩放由 `size` 和人物身高共同决定：

参考：`src/js/shot-generator/components/Three/Attachable.js:302`。

```js
let scale = sceneObject.size * (character.height / 1.8)
```

其中 `1.8` 是默认人物高度。`baby` 和 `child` 有额外修正，避免缩放过度。

## 点击与拖拽交互

### 1. 点击配件

`InteractionManager` 命中 `userData.type === 'attachable'` 时：

- 调用 `selectAttachable({ id, bindId })`。
- 设置 `dragTarget`，后续鼠标移动会拖拽该配件。
- 不走普通对象选择逻辑。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:223`。

```js
if(target.userData && target.userData.type === 'attachable') {
  selectAttachable({ id: target.userData.id, bindId: target.userData.bindedId })
  setDragTarget({ target, x, y })
  return
}
```

### 2. 拖拽配件

`useDraggingManager` 针对配件有专门逻辑：

- 拖拽平面以配件世界位置为基准。
- 拖拽得到的是世界坐标。
- 设置位置前先应用父骨骼世界矩阵，设置世界位置后再应用父骨骼逆矩阵，转回骨骼局部空间。

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:24`、`src/js/shot-generator/hooks/use-dragging-manager.js:65`。

```js
let { x, y, z } = intersection.current.clone().sub(offsets.current[target.userData.id])
let parentMatrixWorld = target.parent.matrixWorld
let parentInverseMatrixWorld = target.parent.getInverseMatrixWorld()
target.applyMatrix4(parentMatrixWorld)
target.position.set(x, y, z)
target.updateMatrixWorld(true)
target.applyMatrix4(parentInverseMatrixWorld)
target.updateMatrixWorld(true)

objectChanges.current[target.userData.id] = { x, y, z }
```

拖拽结束后 `updateObjects` 会把 `x/y/z` 写回 Redux。

### 3. 配件旋转

配件选中后会创建 `ObjectRotationControl`，并注册快捷键切换旋转模式。旋转结束时通过 `ref.current.worldQuaternion()` 得到世界旋转并写回 Redux。

参考：`src/js/shot-generator/components/Three/Attachable.js:181`、`src/js/shot-generator/components/Three/Attachable.js:203`。

## 人物姿态/高度变化时如何保持绑定

当人物的 `rotation`、`height`、`posePresetId`、`handPosePresetId` 变化时，`Attachable` 会更新人物世界矩阵，然后调用 `saveToStore()` 保存配件当前世界位置/旋转。

参考：`src/js/shot-generator/components/Three/Attachable.js:296`。

```js
useEffect(() => {
  if(!characterObject.current || !ref.current || !characterLOD || !ref.current.parent) return
  characterObject.current.updateWorldMatrix(false, true)
  saveToStore()
}, [character.rotation, character.height, character.posePresetId, character.handPosePresetId])
```

另外，当拖拽人物本体结束时，`InteractionManager` 会找到所有 `bindedId === 人物 id` 的配件，把它们的世界位置/旋转重新写回 Redux，避免人物移动后配件状态不同步。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:362`。

## 删除联动

删除人物时会自动删除绑定在该人物上的所有配件：

参考：`src/js/shared/reducers/shot-generator.js:974`。

```js
if (draft[id].type === 'character') {
  let attachableIds = Object.values(draft)
    .filter(obj => obj.attachToId === id)
    .map(obj => obj.id)

  for (let attachableId of attachableIds) {
    delete draft[attachableId]
  }
}
```

## 可复用参考代码

如果要在新实现中复刻这个绑定方式，可以抽象成以下流程：

```js
function bindAttachableToCharacterBone({ scene, attachableGroup, characterId, boneName }) {
  const characterObject = scene.__interaction.find(
    object => object.userData.id === characterId
  )
  if (!characterObject) return false

  const skinnedMesh = characterObject.getObjectByProperty('type', 'SkinnedMesh')
  if (!skinnedMesh) return false

  const bone = skinnedMesh.skeleton.getBoneByName(boneName)
  if (!bone) return false

  bone.add(attachableGroup)
  attachableGroup.updateWorldMatrix(true, true)
  return true
}
```

如果需要把“世界空间位置”保存到 store，可以参考：

```js
function getAttachableWorldTransform(attachableGroup) {
  const position = new THREE.Vector3()
  const quaternion = new THREE.Quaternion()
  const scale = new THREE.Vector3()

  const worldMatrix = attachableGroup.matrix.clone()
  worldMatrix.premultiply(attachableGroup.parent.matrixWorld)
  worldMatrix.decompose(position, quaternion, scale)

  const rotation = new THREE.Euler().setFromQuaternion(quaternion, 'XYZ')

  return {
    x: position.x,
    y: position.y,
    z: position.z,
    rotation: {
      x: rotation.x,
      y: rotation.y,
      z: rotation.z
    }
  }
}
```

如果需要把 store 中的“世界空间位置”应用回骨骼子节点，可以参考：

```js
function applyWorldPositionToBoneChild(attachableGroup, worldPosition) {
  const parentMatrixWorld = attachableGroup.parent.matrixWorld
  const parentInverseMatrixWorld = attachableGroup.parent.getInverseMatrixWorld()

  attachableGroup.applyMatrix4(parentMatrixWorld)
  attachableGroup.position.set(worldPosition.x, worldPosition.y, worldPosition.z)
  attachableGroup.updateMatrixWorld(true)
  attachableGroup.applyMatrix4(parentInverseMatrixWorld)
  attachableGroup.updateMatrixWorld(true)
}
```

## 注意点

- 代码中使用的是 `bindedId`，不是标准英文 `boundId`；这是现有字段名，交互代码依赖它。
- `attachToId` 是数据层字段，`bindedId` 是 Three 对象 `userData` 字段，两者指向同一个人物 id。
- 当前实现假设目标人物中存在 `SkinnedMesh`，且 skeleton 中存在 `bindBone` 指定的骨骼；新增模型时需要保证骨骼命名一致。
- `sceneObject.status === 'PENDING'` 只用于首次初始化，完成后会被更新为 `Loaded`。
- 用户自定义模型与内置模型的初始定位逻辑不同：自定义非头发模型默认放到骨骼世界位置，内置模型使用模型预设的 `x/y/z/rotation`。
