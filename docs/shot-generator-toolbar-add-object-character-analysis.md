# Shot Generator 顶部菜单添加物体/人物实现分析

## 背景

Shot Generator 页面顶部菜单中的“物体”“人物”按钮来自 `Toolbar` 组件。用户点击按钮后，并不是直接向 Three.js 场景里手动 `scene.add()`，而是通过 Redux action 创建一个 `sceneObject` 数据对象。主场景组件监听 Redux 状态变化，再把新的 `sceneObject` 渲染成 React Three Fiber 中的 `<ModelObject />` 或 `<Character />`。

整体链路可以概括为：

```text
顶部 Toolbar 点击
  -> 读取当前激活相机状态并构造 THREE.PerspectiveCamera
  -> 根据当前相机方向计算新对象落点
  -> dispatch CREATE_OBJECT
  -> sceneObjects reducer 写入 Redux 状态
  -> SceneManagerR3fLarge 按 type 过滤 sceneObjects
  -> React Three Fiber 渲染 ModelObject / Character 到主场景
  -> dispatch SELECT_OBJECT，使新对象成为选中状态
```

## 关键代码位置

| 环节 | 文件 | 说明 |
| --- | --- | --- |
| Shot Generator 页面装配 | `src/js/shot-generator/components/Editor/index.js:191` | 将 `Toolbar` 放到页面顶部；主场景 Canvas 使用 `SceneManagerR3fLarge` 渲染。 |
| 顶部按钮 UI 与点击事件 | `src/js/shot-generator/components/Toolbar/index.js:147`、`src/js/shot-generator/components/Toolbar/index.js:156`、`src/js/shot-generator/components/Toolbar/index.js:258` | “物体”“人物”按钮绑定点击处理函数。 |
| 创建默认物体/人物数据 | `src/js/shared/actions/scene-object-creators.js:73`、`src/js/shared/actions/scene-object-creators.js:93` | 生成默认 `object` / `character` 的 action payload。 |
| 计算新对象位置与朝向 | `src/js/shared/actions/scene-object-creators.js:6` | 基于当前相机朝向，默认放在相机前方约 5 米，并朝向相机。 |
| Redux 写入 sceneObjects | `src/js/shared/reducers/shot-generator.js:937` | `CREATE_OBJECT` 将 payload 写入 `state.sceneObjects[id]`。 |
| 自动生成显示名 | `src/js/shared/reducers/shot-generator.js:340` | `withDisplayNames` 根据 `model/name/type` 生成如 `Box 1`、`Adult-male 1` 的名字。 |
| 主场景按类型渲染 | `src/js/shot-generator/SceneManagerR3fLarge.js:138`、`src/js/shot-generator/SceneManagerR3fLarge.js:365`、`src/js/shot-generator/SceneManagerR3fLarge.js:380` | 从 `sceneObjects` 过滤 object/character 并分别渲染。 |
| 物体 Three 组件 | `src/js/shot-generator/components/Three/ModelObject.js:44` | 将 `type: object` 的数据转成 Three group/mesh。 |
| 人物 Three 组件 | `src/js/shot-generator/components/Three/Character.js:43` | 加载人物 glTF、骨骼、IK 和选择辅助。 |
| 模型路径解析 | `src/js/services/model-loader.js:71` | 根据 `type + model` 解析内置或项目模型文件路径。 |
| 中文按钮文案 | `src/js/locales/zh-CN.json:504`、`src/js/locales/zh-CN.json:511` | “物体”“人物”标题和 tooltip 文案。 |

## 页面结构与 Toolbar 入口

`Editor` 是 Shot Generator 的主 UI 容器。顶部菜单和主场景 Canvas 都在这里组合：

- `src/js/shot-generator/components/Editor/index.js:191` 渲染 `<Toolbar />`。
- `src/js/shot-generator/components/Editor/index.js:244` 渲染 `<SceneManagerR3fLarge />`，也就是右侧主视图中的 3D 场景。
- `Toolbar` 收到 `withState`、`ipcRenderer`、`notifications` 等 props，其中本次关注的是 `withState`，它允许 Toolbar 在点击时读取当前 Redux state。

相关代码：

```js
<Toolbar
  withState={withState}
  ipcRenderer={ipcRenderer}
  notifications={notifications}
/>
```

## Toolbar 如何响应点击

`Toolbar` 通过 `connect` 注入 Redux action creator：

- `createModelObject: SceneObjectCreators.createModelObject`
- `createCharacter: SceneObjectCreators.createCharacter`
- `selectObject`
- `undoGroupStart`
- `undoGroupEnd`

位置：`src/js/shot-generator/components/Toolbar/index.js:34`

### “物体”点击流程

位置：`src/js/shot-generator/components/Toolbar/index.js:147`

```js
const onCreateObjectClick = () => {
  let id = THREE.Math.generateUUID()
  initCamera()
  undoGroupStart()
  createModelObject(id, camera.current, room.visible && roomObject3d)
  selectObject(id)
  undoGroupEnd()
}
```

这段逻辑做了 5 件事：

1. 用 `THREE.Math.generateUUID()` 生成新对象 id。
2. 调用 `initCamera()`，把 Redux 中当前激活相机的状态还原成一个 Three.js `PerspectiveCamera`。
3. 开启 undo 分组，保证“创建 + 选中”可以作为一次撤销操作处理。
4. 调用 `createModelObject(...)`，派发 `CREATE_OBJECT`。
5. 调用 `selectObject(id)`，让新添加的物体立即成为当前选中对象。

### “人物”点击流程

位置：`src/js/shot-generator/components/Toolbar/index.js:156`

```js
const onCreateCharacterClick = () => {
  let id = THREE.Math.generateUUID()
  initCamera()
  undoGroupStart()
  createCharacter(id, camera.current, room.visible && roomObject3d)
  selectObject(id)
  undoGroupEnd()
}
```

人物和物体的点击流程基本一致，差异只在调用的 creator 是 `createCharacter`，最终 payload 的 `type`、默认模型、骨骼/姿态字段不同。

### 按钮 JSX 绑定

位置：`src/js/shot-generator/components/Toolbar/index.js:258`

```js
<a href="#"
   onClick={preventDefault(onCreateObjectClick)}
   {...objectTooltipEvents}>
  <Icon src="icon-toolbar-object"/>
  <span>{t("shot-generator.toolbar.object.title")}</span>
</a>
<a href="#"
   onClick={preventDefault(onCreateCharacterClick)}
   {...characterTooltipEvents}>
  <Icon src="icon-toolbar-character"/>
  <span>{t("shot-generator.toolbar.character.title")}</span>
</a>
```

`preventDefault` 只是阻止 `<a href="#">` 改变页面 hash，然后执行真实点击函数。

## 当前相机如何参与新对象定位

点击时先执行 `initCamera()`，位置：`src/js/shot-generator/components/Toolbar/index.js:120`。

它做的事情是：

1. 通过 `withState` 读取当前 Redux state。
2. 用 `getActiveCamera(state)` 找到当前激活相机 id。
3. 用 `getSceneObjects(state)[activeCameraId]` 取出相机数据。
4. 把相机数据还原到 `camera.current` 这个 `THREE.PerspectiveCamera` 上。

关键坐标映射：

```js
camera.current.position.set(cameraState.x, cameraState.z, cameraState.y)
camera.current.rotation.set(cameraState.tilt, cameraState.rotation, cameraState.roll)
```

注意：项目数据中的 `y/z` 和 Three.js 坐标系有转换关系。业务对象里常见的是 `{ x, y, z }`，渲染到 Three.js 时通常会用 `[x, z, y]`。

## 默认位置与朝向如何计算

`createModelObject` 和 `createCharacter` 都会调用 `generatePositionAndRotation(camera, room)`。

位置：`src/js/shared/actions/scene-object-creators.js:6`

核心逻辑：

1. 获取当前相机世界朝向：`camera.getWorldDirection(direction)`。
2. 默认距离为 `5` 米。
3. 如果房间可见，则用 raycaster 朝相机前方检测房间墙面；如果墙面距离小于 5 米，则把对象放在相机到墙面的中点，避免穿墙。
4. 计算相机前方落点：`camera.position + direction * distance`。
5. 将新对象放到地面高度 `y = 0`。
6. 在 x/z 方向随机偏移约 `+-0.3m`，避免连续添加对象完全重叠。
7. 调用 `obj.lookAt(camera.position)`，让对象朝向当前相机。
8. 返回业务坐标格式 `{ x, y, z, rotation }`。

关键代码位置：`src/js/shared/actions/scene-object-creators.js:28`

```js
let center = new THREE.Vector3().addVectors(
  camera.position,
  direction.multiplyScalar(distance)
)

let obj = new THREE.Object3D()
obj.position.set(center.x, 0, center.z)
obj.position.x += (Math.random() * 2 - 1) * 0.3
obj.position.z += (Math.random() * 2 - 1) * 0.3
obj.lookAt(camera.position)
```

所以，从用户体验上看，“点击物体/人物后出现在主场景中”，实际是“出现在当前相机面前约 5 米处，并朝向相机”。

## 物体对象的数据结构

位置：`src/js/shared/actions/scene-object-creators.js:73`

`createModelObject` 创建的是默认 box：

```js
return createObject({
  id,
  type: 'object',
  model: 'box',
  tintColor: '#000000',
  width: 1, height: 1, depth: 1,
  x, y, z,
  rotation: { x: 0, y: rotation, z: 0 },
  visible: true
})
```

重点字段：

- `type: 'object'`：决定后续由 `ModelObject` 组件渲染。
- `model: 'box'`：默认物体是内置 box；`ModelObject` 对 `box` 有特殊处理，不走 glTF 加载。
- `width/height/depth`：默认尺寸 1 米。
- `rotation`：物体 rotation 是对象格式 `{ x, y, z }`。
- `visible: true`：创建后立即可见。

## 人物对象的数据结构

位置：`src/js/shared/actions/scene-object-creators.js:93`

`createCharacter` 创建默认男性角色：

```js
return createObject({
  id,
  type: 'character',
  height: 1.8,
  model: 'adult-male',
  x, y, z,
  rotation,
  headScale: 1,
  tintColor: '#000000',
  morphTargets: {
    mesomorphic: 0,
    ectomorphic: 0,
    endomorphic: 0
  },
  posePresetId: defaultPosePreset.id,
  skeleton: defaultPosePreset.state.skeleton,
  visible: true
})
```

重点字段：

- `type: 'character'`：决定后续由 `Character` 组件渲染。
- `model: 'adult-male'`：默认人物模型，对应内置 `adult-male.glb`。
- `height: 1.8`：默认身高 1.8 米。
- `rotation`：人物 rotation 是单个 y 轴角度，不是 `{ x, y, z }`。
- `posePresetId` / `skeleton`：新人物默认使用站立姿态 preset。
- `morphTargets`：用于体型 morph 控制。

## Redux 如何把对象加入场景状态

`SceneObjectCreators.createModelObject/createCharacter` 最终都返回 `createObject(...)` action creator。

位置：`src/js/shared/reducers/shot-generator.js:1721`

```js
createObject: values => ({ type: 'CREATE_OBJECT', payload: values })
```

真正写入状态的位置是 `sceneObjectsReducer` 的 `CREATE_OBJECT` 分支。

位置：`src/js/shared/reducers/shot-generator.js:937`

```js
case 'CREATE_OBJECT':
  let id = action.payload.id != null
    ? action.payload.id
    : THREE.Math.generateUUID()

  draft[id] = {
    ...action.payload, id
  }

  return withDisplayNames(draft)
```

这里的关键点：

- `sceneObjects` 是以 id 为 key 的对象字典。
- 新对象并不是直接 Three.js 实例，而是普通 JSON 风格的 scene object 数据。
- 写入后调用 `withDisplayNames(draft)`，刷新左侧元素列表和场景标注中使用的 `displayName`。

## 主场景如何渲染新增对象

`SceneManagerR3fLarge` 从 Redux `sceneObjects` 中按类型拆分 id。

位置：`src/js/shot-generator/SceneManagerR3fLarge.js:138`

```js
const modelObjectIds = useMemo(() => {
  return Object.values(sceneObjects).filter(o => o.type === 'object').map(o => o.id)
}, [sceneObjectLength])

const characterIds = useMemo(() => {
  return Object.values(sceneObjects).filter(o => o.type === 'character').map(o => o.id)
}, [sceneObjectLength])
```

### 物体渲染

位置：`src/js/shot-generator/SceneManagerR3fLarge.js:365`

```js
modelObjectIds.map(id => {
  let sceneObject = sceneObjects[id]
  return <ModelObject
    path={ModelLoader.getFilepathForModel(sceneObject, {storyboarderFilePath})}
    sceneObject={sceneObject}
    isSelected={selections.includes(sceneObject.id)}
    updateObject={updateObject}
    objectRotationControl={objectRotationControl.current}
  />
})
```

`ModelObject` 内部逻辑在 `src/js/shot-generator/components/Three/ModelObject.js:44`。

- 如果 `sceneObject.model === 'box'`，直接创建 rounded box geometry，不加载 glTF。
- 如果不是 box，则通过 `useAsset(path)` 加载 glTF 模型。
- Three.js group 的 `userData` 写入 `{ type: 'object', id, locked, blocked }`，后续选择、拖拽、控制器都靠它识别。
- 渲染坐标使用 `position={[x, z, y]}`，再次体现业务坐标和 Three 坐标的 y/z 转换。

相关位置：`src/js/shot-generator/components/Three/ModelObject.js:51`、`src/js/shot-generator/components/Three/ModelObject.js:143`

### 人物渲染

位置：`src/js/shot-generator/SceneManagerR3fLarge.js:380`

```js
characterIds.map(id => {
  let sceneObject = sceneObjects[id]
  return <Character
    path={ModelLoader.getFilepathForModel(sceneObject, {storyboarderFilePath})}
    sceneObject={sceneObject}
    modelSettings={models[sceneObject.model]}
    isSelected={selections.includes(id)}
    selectedBone={selectedBone}
    updateCharacterSkeleton={updateCharacterSkeleton}
    updateCharacterIkSkeleton={updateCharacterIkSkeleton}
    updateObject={updateObject}
    objectRotationControl={objectRotationControl.current}
  />
})
```

`Character` 内部逻辑在 `src/js/shot-generator/components/Three/Character.js:43`。

- 使用 `useAsset(path)` 加载人物 glTF。
- 通过 `cloneGltf` 克隆模型，避免多个人物共享同一实例。
- 查找 `SkinnedMesh`，构建 `THREE.LOD`。
- 根据 `sceneObject.skeleton`、`handSkeleton`、`morphTargets`、`headScale` 更新角色状态。
- 选中人物时初始化 `BonesHelper`、`SGIkHelper`、`ObjectRotationControl`，用于骨骼/IK/旋转控制。
- group 的 `userData` 写入 `{ type: 'character', id, height, locked, blocked, model, name }`。
- 渲染坐标同样使用 `position={[x, z, y]}`，人物旋转使用 `rotation={[0, rotation, 0]}`。

相关位置：`src/js/shot-generator/components/Three/Character.js:52`、`src/js/shot-generator/components/Three/Character.js:354`、`src/js/shot-generator/components/Three/Character.js:440`

## 模型文件路径如何解析

主场景传入组件前，会调用：

```js
ModelLoader.getFilepathForModel(sceneObject, { storyboarderFilePath })
```

位置：`src/js/services/model-loader.js:71`

规则如下：

- 内置物体：`type: 'object'` -> `data/shot-generator/objects/<model>.glb`
- 内置人物：`type: 'character'` -> `data/shot-generator/dummies/gltf/<model>.glb`
- 自定义模型：如果 `model` 是带目录和扩展名的路径，则按绝对路径或项目相对路径解析。

因此默认人物 `model: 'adult-male'` 会解析到：

```text
data/shot-generator/dummies/gltf/adult-male.glb
```

默认物体 `model: 'box'` 虽然也能构造路径，但 `ModelObject` 在 `sceneObject.model === 'box'` 时不会加载该路径，而是直接生成内置 rounded box geometry。

## 选中状态与左侧元素列表的关系

点击添加后，Toolbar 会立刻调用 `selectObject(id)`。

位置：`src/js/shared/reducers/shot-generator.js:810`

`selectionsReducer` 会把当前选择替换成新对象 id：

```js
case 'SELECT_OBJECT':
  return (action.payload == null)
    ? []
    : Array.isArray(action.payload) ? action.payload : [action.payload]
```

主场景渲染时通过 `selections.includes(id)` 给 `ModelObject` / `Character` 传入 `isSelected`。组件内部再调用 outline 和控制器逻辑显示选中效果。

- 物体选中效果：`src/js/shot-generator/components/Three/ModelObject.js:94`
- 人物选中效果：`src/js/shot-generator/components/Three/Character.js:354`

这也是为什么新建后左侧列表和主场景里会立即选中该对象。

## 物体和人物添加方式的差异总结

| 对比项 | 物体 | 人物 |
| --- | --- | --- |
| 点击函数 | `onCreateObjectClick` | `onCreateCharacterClick` |
| creator | `createModelObject` | `createCharacter` |
| 默认 type | `object` | `character` |
| 默认 model | `box` | `adult-male` |
| 默认尺寸 | `width/height/depth = 1` | `height = 1.8` |
| rotation 数据 | `{ x, y, z }` | 单个 y 轴角度 |
| 渲染组件 | `ModelObject` | `Character` |
| 模型加载 | box 直接生成 geometry，其他 object 可加载 glTF | 默认加载 `adult-male.glb`，包含骨骼/LOD/IK |
| 选中控制 | 物体旋转控制 | 人物旋转 + 骨骼 + IK 控制 |

## 技术实现特点

1. **数据驱动，而非命令式 scene.add**：点击只创建 Redux 数据，Three 场景由 React Three Fiber 根据状态自动渲染。
2. **对象定位依赖当前相机**：新对象出现在当前激活相机前方，并朝向相机。
3. **业务坐标与 Three 坐标有转换**：存储时是 `{ x, y, z }`，渲染时多处使用 `[x, z, y]`。
4. **Undo 友好**：创建和选中包裹在 `undoGroupStart/undoGroupEnd` 中，方便一次撤销。
5. **类型决定渲染组件**：`type: 'object'` 走 `ModelObject`，`type: 'character'` 走 `Character`。
6. **默认模型只是起点**：添加后用户可在左侧属性面板更换模型、尺寸、颜色、姿态等字段，最终仍然是更新同一个 `sceneObject`。

## 调试建议

如果后续要调试“点击后没有添加到主场景”的问题，建议按下面顺序排查：

1. 在 `src/js/shot-generator/components/Toolbar/index.js:147` 或 `src/js/shot-generator/components/Toolbar/index.js:156` 打断点，确认点击事件是否触发。
2. 在 `src/js/shared/actions/scene-object-creators.js:73` 或 `src/js/shared/actions/scene-object-creators.js:93` 查看 payload 是否生成正确。
3. 在 `src/js/shared/reducers/shot-generator.js:937` 确认 `CREATE_OBJECT` 是否写入 `sceneObjects`。
4. 在 `src/js/shot-generator/SceneManagerR3fLarge.js:138` 确认新 id 是否进入 `modelObjectIds` 或 `characterIds`。
5. 在 `src/js/shot-generator/components/Three/ModelObject.js:44` 或 `src/js/shot-generator/components/Three/Character.js:43` 确认组件是否被挂载。
6. 如果是人物不显示，重点检查 `src/js/services/model-loader.js:71` 解析出的 glTF 路径和 `useAsset(path)` 加载状态。
