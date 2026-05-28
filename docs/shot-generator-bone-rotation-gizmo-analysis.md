# Shot Generator 骨骼三轴圆环旋转控制器实现分析

本文分析 Shot Generator 右侧主场景中，编辑人物骨骼时出现的“三轴圆环控制器”（截图红框中的紫色旋转环）。该控制器不是单独的 UI 组件，而是由 `ObjectRotationControl` 封装自定义 `TransformControls`，再绑定到当前选中的 `THREE.Bone` 上实现的。

## 总体结论

右侧主场景的人物骨骼旋转控制器由以下链路组成：

1. `InteractionManager` 负责鼠标命中骨骼或已显示的旋转 gizmo，并把选中的骨骼 uuid 写入 Redux 的 `selectedBone`。
2. `Character` 监听 `selectedBone`，找到对应的 `THREE.Bone`，调用 `boneRotationControl.selectObject(bone, selectedBone)` 把旋转控制器挂到骨骼上。
3. `ObjectRotationControl` 创建并配置 `TransformControls`：只允许旋转、使用本地坐标、默认大小 `0.2`、类型标记为 `objectControl`。
4. `TransformControlsGizmo` 用 `THREE.TorusBufferGeometry` 分别生成 X/Y/Z 三个圆环；在当前项目中三个轴都使用紫色系材质，因此画面中看到的是多层紫色圆环。
5. 鼠标拖动某个圆环时，`TransformControls.pointerMove` 根据鼠标在隐藏平面上的位移计算旋转轴与角度，直接修改目标骨骼的 `quaternion`。
6. 鼠标释放时，`ObjectRotationControl.onMouseUp` 调用 `updateCharacterSkeleton`，把骨骼名和欧拉角 rotation 写回 Redux 的 `sceneObject.skeleton`。

## 关键代码文件

- 主场景挂载入口：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 人物模型、骨骼选择与骨骼旋转控制器挂载：`src/js/shot-generator/components/Three/Character.js`
- 主场景鼠标命中与骨骼选择：`src/js/shot-generator/components/Three/InteractionManager.js`
- 骨骼可视化与辅助命中：`src/js/xr/src/three/BonesHelper.js`
- 旋转控制器封装：`src/js/shared/IK/objects/ObjectRotationControl.js`
- 三轴圆环 gizmo 与拖拽数学：`src/js/shared/IK/utils/TransformControls.js`
- Redux 选择状态与骨骼 rotation 持久化：`src/js/shared/reducers/shot-generator.js`

## 入口：主场景把 selectedBone 传给 Character

`SceneManagerR3fLarge` 从 Redux 读取 `selectedBone`：

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:75`、`src/js/shot-generator/SceneManagerR3fLarge.js:82`。

```js
const SceneManagerR3fLarge = connect(
  state => ({
    selectedBone: getSelectedBone(state),
  })
)
```

随后在渲染每个 Character 时，把 `selectedBone` 和 `updateCharacterSkeleton` 传入：

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:390`、`src/js/shot-generator/SceneManagerR3fLarge.js:396`、`src/js/shot-generator/SceneManagerR3fLarge.js:397`、`src/js/shot-generator/SceneManagerR3fLarge.js:402`。

```jsx
<Character
  selectedBone={ selectedBone }
  updateCharacterSkeleton={ updateCharacterSkeleton }
  objectRotationControl={ objectRotationControl.current }
/>
```

这里的 `objectRotationControl` 是人物整体绕 Y 轴旋转用的；真正骨骼三轴圆环是在 `Character` 内部另建的 `boneRotationControl`。

## 鼠标如何选中骨骼

### 点击什么位置会触发圆环显示

触发骨骼三轴圆环显示的操作不是“点击人物皮肤网格任意位置”，而是：

1. **先选中人物对象**：例如截图中的 `Adult-male 1`。人物选中后，`Character` 才会把 `BonesHelper` 添加到人物 group 中。
2. **再点击人物身上的骨骼辅助体 / 骨骼段**：也就是选中人物后叠加显示出来的骨骼 helper，而不是普通的 skinned mesh 表面。
3. **射线命中 `BonesHelper` 后选择骨骼**：`InteractionManager` 会执行 `raycaster.current.intersectObject(BonesHelper.getInstance())`，命中后用 `hits[0].bone.uuid` 更新 Redux 的 `selectedBone`。
4. **`Character` 根据 `selectedBone` 显示圆环**：`Character` 在 skeleton 中按 uuid 找到对应的 `THREE.Bone`，再调用 `boneRotationControl.current.selectObject(bone, selectedBone)`，圆环控制器随即 attach 到该骨骼并显示。

因此，实际可点击区域可以理解为“人物被选中后可见/可命中的骨骼辅助段”。如果只点到人物模型的皮肤表面，但没有命中 `BonesHelper` 或已显示的 `objectControl`，通常不会显示骨骼三轴圆环。

对应代码链路：

- 人物选中后添加骨骼 helper：`src/js/shot-generator/components/Three/Character.js:358`、`src/js/shot-generator/components/Three/Character.js:359`、`src/js/shot-generator/components/Three/Character.js:365`
- 主场景点击后检测骨骼 helper：`src/js/shot-generator/components/Three/InteractionManager.js:298`、`src/js/shot-generator/components/Three/InteractionManager.js:299`
- 命中后写入 `selectedBone`：`src/js/shot-generator/components/Three/InteractionManager.js:301`、`src/js/shot-generator/components/Three/InteractionManager.js:306`
- `Character` 监听 `selectedBone` 并 attach 控制器：`src/js/shot-generator/components/Three/Character.js:340`、`src/js/shot-generator/components/Three/Character.js:342`、`src/js/shot-generator/components/Three/Character.js:346`

简化流程如下：

```text
选中人物
  ↓
BonesHelper 显示/参与命中
  ↓
点击骨骼辅助段
  ↓
selectBone(hits[0].bone.uuid)
  ↓
Character 找到对应 THREE.Bone
  ↓
boneRotationControl.selectObject(bone, selectedBone)
  ↓
三轴圆环控制器显示
```

还有一种“再次触发/保持显示”的路径：如果圆环已经显示出来，用户点击圆环本身，`InteractionManager` 会把它识别为 `objectControl`。当 `objectControl.object` 是 `THREE.Bone` 时，也会重新执行 `selectBone(selectedObjectControl.uuid)`，从而保持或刷新当前骨骼控制器。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:235`、`src/js/shot-generator/components/Three/InteractionManager.js:239`、`src/js/shot-generator/components/Three/InteractionManager.js:291`、`src/js/shot-generator/components/Three/InteractionManager.js:292`。

### 1. BonesHelper 生成可点击的骨骼辅助体

选中人物后，`Character` 会初始化并添加 `BonesHelper`，让人物骨架在主场景中可视化、可被射线命中：

参考：`src/js/shot-generator/components/Three/Character.js:358`、`src/js/shot-generator/components/Three/Character.js:359`、`src/js/shot-generator/components/Three/Character.js:365`。

```js
if (isSelected) {
  BonesHelper.getInstance().initialize(lod.children[0])
  ref.current.add(BonesHelper.getInstance())
}
```

`BonesHelper.initialize` 会遍历 skinned mesh 的 `skeleton.bones`，跳过没有子骨骼的末端骨骼，为每根可显示骨骼从对象池取一个 helper mesh，并建立 helper 与原始 `THREE.Bone` 的映射：

参考：`src/js/xr/src/three/BonesHelper.js:39`、`src/js/xr/src/three/BonesHelper.js:58`、`src/js/xr/src/three/BonesHelper.js:62`、`src/js/xr/src/three/BonesHelper.js:65`、`src/js/xr/src/three/BonesHelper.js:73`、`src/js/xr/src/three/BonesHelper.js:78`。

```js
let bones = skinnedMesh.skeleton.bones
for (let i = 0, n = bones.length; i < n; i++) {
  bone = bones[i]
  if (bone.children.length === 0) continue
  helpingBone = this.helperBonesPool.takeBone()
  this.helpingBonesRelation.push({ helpingBone, originalBone: bone })
}
```

`BonesHelper.raycast` 会把射线命中的 helper 转回原始骨骼，写到 `result.bone` 上：

参考：`src/js/xr/src/three/BonesHelper.js:172`、`src/js/xr/src/three/BonesHelper.js:179`、`src/js/xr/src/three/BonesHelper.js:182`。

```js
let results = raycaster.intersectObjects(this.bonesGroup.children)
result.bone = this.helpingBonesRelation
  .find(object => object.helpingBone.id === result.object.id)
  .originalBone
```

### 2. InteractionManager 处理点击骨骼

主场景的 pointer 事件绑定在 canvas 上：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:437`、`src/js/shot-generator/components/Three/InteractionManager.js:438`、`src/js/shot-generator/components/Three/InteractionManager.js:441`。

```js
activeGL.domElement.addEventListener('pointerdown', onPointerDown)
window.addEventListener('pointerup', onPointerUp)
```

当当前选中对象是人物，并且再次点击该人物时，`InteractionManager` 会用 `raycaster.intersectObject(BonesHelper.getInstance())` 检查是否点到了骨骼 helper。命中后调用 `selectBone(hits[0].bone.uuid)`：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:275`、`src/js/shot-generator/components/Three/InteractionManager.js:298`、`src/js/shot-generator/components/Three/InteractionManager.js:299`、`src/js/shot-generator/components/Three/InteractionManager.js:301`、`src/js/shot-generator/components/Three/InteractionManager.js:306`。

```js
raycaster.current.setFromCamera({ x, y }, camera)
let hits = raycaster.current.intersectObject(BonesHelper.getInstance())
if (!isSelectedControlPoint && hits.length) {
  selectBone(hits[0].bone.uuid)
  setDragTarget({ target, x, y })
  return
}
```

当用户直接点击已经显示出来的骨骼旋转圆环时，命中对象会被识别为 `objectControl`，如果 `targetElement.type === "Bone"`，同样会调用 `selectBone(selectedObjectControl.uuid)`：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:235`、`src/js/shot-generator/components/Three/InteractionManager.js:239`、`src/js/shot-generator/components/Three/InteractionManager.js:291`、`src/js/shot-generator/components/Three/InteractionManager.js:292`。

```js
if (targetElement.type === "Bone") {
  selectedObjectControl = targetElement
}

if (!isSelectedControlPoint && selectedObjectControl) {
  selectBone(selectedObjectControl.uuid)
  setDragTarget({ target, x, y, isObjectControl: true })
  return
}
```

## Redux 如何保存当前选中的骨骼

`selectedBone` 是 Redux undoable present state 的一部分：

参考：`src/js/shared/reducers/shot-generator.js:33`。

```js
const getSelectedBone = state => state.undoable.present.selectedBone
```

`selectBone` action creator 很直接，payload 就是骨骼 uuid 或 `null`：

参考：`src/js/shared/reducers/shot-generator.js:1717`。

```js
selectBone: id => ({ type: 'SELECT_BONE', payload: id })
```

`selectedBoneReducer` 会在选择普通对象、加载场景、XR 更新、删除对象时清空骨骼选择；收到 `SELECT_BONE` 时保存 uuid：

参考：`src/js/shared/reducers/shot-generator.js:1270`、`src/js/shared/reducers/shot-generator.js:1273`、`src/js/shared/reducers/shot-generator.js:1279`、`src/js/shared/reducers/shot-generator.js:1283`。

```js
case 'SELECT_OBJECT':
  return null
case 'SELECT_BONE':
  return action.payload
```

## Character 如何显示骨骼旋转圆环

### 1. 创建骨骼专用 ObjectRotationControl

`Character` 内部有一个 `boneRotationControl` ref：

参考：`src/js/shot-generator/components/Three/Character.js:64`。

```js
const boneRotationControl = useRef(null)
```

首次拿到人物 group 后创建 `ObjectRotationControl`，并把保存回 Redux 的回调设置为 `updateCharacterSkeleton`：

参考：`src/js/shot-generator/components/Three/Character.js:405`、`src/js/shot-generator/components/Three/Character.js:407`、`src/js/shot-generator/components/Three/Character.js:408`、`src/js/shot-generator/components/Three/Character.js:411`、`src/js/shot-generator/components/Three/Character.js:412`。

```js
boneRotationControl.current = new ObjectRotationControl(scene.children[0], camera, gl.domElement, ref.current.uuid)
boneRotationControl.current.setCharacterId(ref.current.uuid)
boneRotationControl.current.setUpdateCharacter((name, rotation) => {
  updateCharacterSkeleton({ id: sceneObject.id, name, rotation })
})
```

这段是骨骼旋转控制器和 Redux 持久化之间的核心连接：控制器释放鼠标后拿到 `this.object.name` 和 `this.object.rotation`，这里把它们转换成 `UPDATE_CHARACTER_SKELETON`。

### 2. selectedBone 改变时 attach 到 THREE.Bone

`Character` 监听 `selectedBone`。如果有旧选择，先重置 `BonesHelper` 的高亮；如果有新的 `selectedBone`，就在当前 skeleton 里按 uuid 找到 `THREE.Bone`，高亮 helper，并把旋转控制器 attach 到该骨骼：

参考：`src/js/shot-generator/components/Three/Character.js:332`、`src/js/shot-generator/components/Three/Character.js:336`、`src/js/shot-generator/components/Three/Character.js:340`、`src/js/shot-generator/components/Three/Character.js:342`、`src/js/shot-generator/components/Three/Character.js:345`、`src/js/shot-generator/components/Three/Character.js:346`、`src/js/shot-generator/components/Three/Character.js:350`。

```js
if (selectedBone) {
  let bone = skeleton.bones.find(object => object.uuid === selectedBone)
  if (bone) {
    BonesHelper.getInstance().selectBone(bone)
    boneRotationControl.current.selectObject(bone, selectedBone)
  }
} else {
  boneRotationControl.current.deselectObject()
}
```

当渲染目标 `activeGL` 改变时，也会重新把控制器绑定到当前选中的骨骼，避免 renderer/canvas 切换后事件仍绑在旧 domElement 上：

参考：`src/js/shot-generator/components/Three/Character.js:278`、`src/js/shot-generator/components/Three/Character.js:281`、`src/js/shot-generator/components/Three/Character.js:283`、`src/js/shot-generator/components/Three/Character.js:285`。

```js
boneRotationControl.current.control.domElement = activeGL.domElement
let bone = skeleton.bones.find(object => object.uuid === selectedBone)
if (bone) boneRotationControl.current.selectObject(bone, selectedBone)
```

## ObjectRotationControl：骨骼三轴控制器的封装层

`ObjectRotationControl` 在构造函数中创建 `TransformControls`，并强制设置为“仅旋转 + 本地坐标”：

参考：`src/js/shared/IK/objects/ObjectRotationControl.js:4`、`src/js/shared/IK/objects/ObjectRotationControl.js:6`、`src/js/shared/IK/objects/ObjectRotationControl.js:7`、`src/js/shared/IK/objects/ObjectRotationControl.js:8`、`src/js/shared/IK/objects/ObjectRotationControl.js:9`、`src/js/shared/IK/objects/ObjectRotationControl.js:10`。

```js
this.control = new TransformControls(camera, domElement)
this.control.rotationOnly = true
this.control.setSpace("local")
this.control.setMode('rotate')
this.control.size = 0.2
```

这解释了截图中的行为：

- 圆环只用于旋转，不显示平移箭头或缩放手柄。
- 旋转基于骨骼本地坐标系，而不是世界坐标系。
- 控制器尺寸较小，并在 `TransformControlsGizmo` 中被限制在最小/最大范围内。

构造时还给控制器及其子节点标记 `userData.type = "objectControl"`，供 `InteractionManager` 判定这是控制器而不是普通模型：

参考：`src/js/shared/IK/objects/ObjectRotationControl.js:11`、`src/js/shared/IK/objects/ObjectRotationControl.js:12`。

```js
this.control.userData.type = "objectControl"
this.control.traverse(child => {
  child.userData.type = "objectControl"
})
```

`selectObject` 是把控制器挂到某个对象上的方法。对骨骼编辑来说，这里的 `object` 就是 `THREE.Bone`，`hitmeshid` 是 `selectedBone` uuid：

参考：`src/js/shared/IK/objects/ObjectRotationControl.js:46`、`src/js/shared/IK/objects/ObjectRotationControl.js:53`、`src/js/shared/IK/objects/ObjectRotationControl.js:55`、`src/js/shared/IK/objects/ObjectRotationControl.js:56`、`src/js/shared/IK/objects/ObjectRotationControl.js:66`、`src/js/shared/IK/objects/ObjectRotationControl.js:68`。

```js
this.offsetObject.add(this.control)
this.control.addToScene()
this.control.attach(object)
this.object = object
this.control.addEventListener("transformMouseDown", this.onMouseDown, false)
this.control.addEventListener("transformMouseUp", this.onMouseUp, false)
```

鼠标释放时保存骨骼 rotation：

参考：`src/js/shared/IK/objects/ObjectRotationControl.js:35`、`src/js/shared/IK/objects/ObjectRotationControl.js:37`、`src/js/shared/IK/objects/ObjectRotationControl.js:38`。

```js
onMouseUp = event => {
  this.updateCharacter && this.updateCharacter(this.object.name, this.object.rotation)
  this.object.isRotated = false
}
```

## TransformControls：圆环本体如何生成

### 1. 轴位掩码

`TransformControls.js` 定义了 X/Y/Z 三个轴的 bit mask，默认显示三轴：

参考：`src/js/shared/IK/utils/TransformControls.js:10`、`src/js/shared/IK/utils/TransformControls.js:16`。

```js
const axis = {
  X_axis: 0x0001,
  Y_axis: 0x0002,
  Z_axis: 0x0004
}

const TransformControls = function (camera, domElement, shownAxis = axis.X_axis | axis.Y_axis | axis.Z_axis) {
```

骨骼旋转控制器创建时没有传入 axis，因此默认显示三轴。人物整体旋转则不同，`Character` 会对外部 `objectRotationControl` 调用 `setShownAxis(axis.Y_axis)`，只显示 Y 轴：

参考：`src/js/shot-generator/components/Three/Character.js:379`、`src/js/shot-generator/components/Three/Character.js:380`。

```js
props.objectRotationControl.selectObject(ref.current, ref.current.uuid, highestPoint)
props.objectRotationControl.control.setShownAxis(axis.Y_axis)
```

### 2. 材质颜色

圆环使用 `MeshBasicMaterial`，关闭深度测试与深度写入，因此它会稳定显示在模型上方，不容易被人物身体遮挡：

参考：`src/js/shared/IK/utils/TransformControls.js:748`、`src/js/shared/IK/utils/TransformControls.js:749`、`src/js/shared/IK/utils/TransformControls.js:750`、`src/js/shared/IK/utils/TransformControls.js:751`、`src/js/shared/IK/utils/TransformControls.js:752`。

```js
var gizmoMaterial = new THREE.MeshBasicMaterial({
  depthTest: false,
  depthWrite: false,
  transparent: true,
  side: THREE.DoubleSide
})
```

X/Y/Z 三个可见圆环的材质是紫色系，而不是 Three.js 默认红绿蓝：

参考：`src/js/shared/IK/utils/TransformControls.js:768`、`src/js/shared/IK/utils/TransformControls.js:769`、`src/js/shared/IK/utils/TransformControls.js:771`、`src/js/shared/IK/utils/TransformControls.js:772`、`src/js/shared/IK/utils/TransformControls.js:774`、`src/js/shared/IK/utils/TransformControls.js:775`。

```js
matX.color.set(0x640AA1)
matY.color.set(0x6D22A1)
matZ.color.set(0x8951B0)
```

这就是截图里三轴圆环看起来都是紫色，只是深浅不同的原因。

### 3. 可见圆环几何体

圆环半径、管径和 picker 容差在 `TransformControlsGizmo` 中定义：

参考：`src/js/shared/IK/utils/TransformControls.js:744`、`src/js/shared/IK/utils/TransformControls.js:745`、`src/js/shared/IK/utils/TransformControls.js:746`、`src/js/shared/IK/utils/TransformControls.js:747`、`src/js/shared/IK/utils/TransformControls.js:889`。

```js
let rotationalGizmoRadius = 1.3
let rotationalGizmoTube = rotationalGizmoRadius / 12
let pickerTolerance = 0.05
rotationalGizmoTube += pickerTolerance
let tubularSegments = 50
```

X/Y/Z 三个可见圆环都通过 `THREE.TorusBufferGeometry` 创建，再用不同初始 rotation 摆到各自轴向：

参考：`src/js/shared/IK/utils/TransformControls.js:892`、`src/js/shared/IK/utils/TransformControls.js:894`、`src/js/shared/IK/utils/TransformControls.js:897`、`src/js/shared/IK/utils/TransformControls.js:899`、`src/js/shared/IK/utils/TransformControls.js:902`、`src/js/shared/IK/utils/TransformControls.js:904`。

```js
gizmoRotate.X = [
  [new THREE.Mesh(new THREE.TorusBufferGeometry(rotationalGizmoRadius, rotationalGizmoTube, 4, tubularSegments), matX), null, [0, -Math.PI / 2, -Math.PI / 2]],
]
gizmoRotate.Y = [
  [new THREE.Mesh(new THREE.TorusBufferGeometry(rotationalGizmoRadius, rotationalGizmoTube, 4, tubularSegments), matY), null, [Math.PI / 2, 0, 0]],
]
gizmoRotate.Z = [
  [new THREE.Mesh(new THREE.TorusBufferGeometry(rotationalGizmoRadius, rotationalGizmoTube, 4, tubularSegments), matZ), null, [0, 0, -Math.PI / 2]],
]
```

### 4. 隐形 picker 圆环

除了可见圆环，代码还创建了对应的 `pickerRotate`。这些 picker 用于射线命中，实际会被隐藏：

参考：`src/js/shared/IK/utils/TransformControls.js:910`、`src/js/shared/IK/utils/TransformControls.js:912`、`src/js/shared/IK/utils/TransformControls.js:918`、`src/js/shared/IK/utils/TransformControls.js:923`、`src/js/shared/IK/utils/TransformControls.js:1103`、`src/js/shared/IK/utils/TransformControls.js:1104`。

```js
pickerRotate.X = [
  [new THREE.Mesh(new THREE.TorusBufferGeometry(rotationalGizmoRadius, rotationalGizmoTube + offset, 4, tubularSegments + pickerTolerance), matRed), null, [0, -Math.PI / 2, -Math.PI / 2]],
]
this.picker["rotate"].visible = false
```

鼠标命中检测并不是检测可见紫色圆环，而是检测隐藏 picker：

参考：`src/js/shared/IK/utils/TransformControls.js:303`、`src/js/shared/IK/utils/TransformControls.js:306`、`src/js/shared/IK/utils/TransformControls.js:308`、`src/js/shared/IK/utils/TransformControls.js:312`。

```js
ray.setFromCamera(pointer, this.camera)
var intersect = ray.intersectObjects(_gizmo.picker[this.mode].children, true)[0] || false
if (intersect) this.axis = intersect.object.name
```

## TransformControls：拖动圆环如何旋转骨骼

### 1. 事件绑定

`ObjectRotationControl.selectObject` 调用 `control.addToScene()` 后，`TransformControls` 会把 pointer 事件绑定到当前 canvas：

参考：`src/js/shared/IK/utils/TransformControls.js:155`、`src/js/shared/IK/utils/TransformControls.js:157`、`src/js/shared/IK/utils/TransformControls.js:158`、`src/js/shared/IK/utils/TransformControls.js:159`。

```js
this.domElement.addEventListener("pointerdown", onPointerDown, false)
this.domElement.addEventListener("pointermove", onPointerHover, false)
document.addEventListener("pointerup", onPointerUp, false)
```

### 2. pointerdown 记录起点状态

按下鼠标时，`TransformControls.pointerDown` 会记录目标对象的起始 position/quaternion/scale、世界坐标与鼠标起始点。这里的目标对象是被 attach 的骨骼：

参考：`src/js/shared/IK/utils/TransformControls.js:322`、`src/js/shared/IK/utils/TransformControls.js:326`、`src/js/shared/IK/utils/TransformControls.js:330`、`src/js/shared/IK/utils/TransformControls.js:359`、`src/js/shared/IK/utils/TransformControls.js:360`、`src/js/shared/IK/utils/TransformControls.js:363`、`src/js/shared/IK/utils/TransformControls.js:365`。

```js
var planeIntersect = ray.intersectObjects([_plane], true)[0] || false
positionStart.copy(this.object.position)
quaternionStart.copy(this.object.quaternion)
this.object.matrixWorld.decompose(worldPositionStart, worldQuaternionStart, worldScaleStart)
pointStart.copy(planeIntersect.point).sub(worldPositionStart)
```

按下后会派发 `pointerdown` 和自定义 `transformMouseDown`：

参考：`src/js/shared/IK/utils/TransformControls.js:369`、`src/js/shared/IK/utils/TransformControls.js:371`、`src/js/shared/IK/utils/TransformControls.js:649`。

```js
this.dragging = true
this.dispatchEvent(mouseDownEvent)
scope.dispatchEvent({ type: "transformMouseDown", value: event })
```

### 3. pointermove 计算旋转轴与角度

拖动时，如果当前模式是 `rotate`，会计算 `offset = pointEnd - pointStart`，并根据当前选中的轴 X/Y/Z 得到 `rotationAxis` 与 `rotationAngle`：

参考：`src/js/shared/IK/utils/TransformControls.js:513`、`src/js/shared/IK/utils/TransformControls.js:515`、`src/js/shared/IK/utils/TransformControls.js:517`、`src/js/shared/IK/utils/TransformControls.js:534`、`src/js/shared/IK/utils/TransformControls.js:536`、`src/js/shared/IK/utils/TransformControls.js:540`、`src/js/shared/IK/utils/TransformControls.js:544`。

```js
} else if (mode === 'rotate') {
  offset.copy(pointEnd).sub(pointStart)
  var ROTATION_SPEED = 20 / worldPosition.distanceTo(_tempVector.setFromMatrixPosition(this.camera.matrixWorld))
  rotationAxis.copy(_unit[axis])
  if (space === 'local') {
    _tempVector.applyQuaternion(worldQuaternion)
  }
  rotationAngle = offset.dot(_tempVector.cross(eye).normalize()) * ROTATION_SPEED
}
```

因为 `ObjectRotationControl` 在构造时设置了 `setSpace("local")`，所以骨骼旋转采用本地空间；这符合骨骼姿态编辑的预期。

### 4. 直接修改骨骼 quaternion

计算角度后，local rotate 分支会从按下时的 `quaternionStart` 开始，乘上当前轴角旋转，最后写回 `object.quaternion`：

参考：`src/js/shared/IK/utils/TransformControls.js:552`、`src/js/shared/IK/utils/TransformControls.js:555`、`src/js/shared/IK/utils/TransformControls.js:557`、`src/js/shared/IK/utils/TransformControls.js:558`。

```js
this.rotationAngle = rotationAngle
if (space === 'local' && axis !== 'E' && axis !== 'XYZE') {
  object.quaternion.copy(quaternionStart)
  object.quaternion.multiply(_tempQuaternion.setFromAxisAngle(rotationAxis, rotationAngle)).normalize()
}
```

这一步是“拖动圆环导致人物肢体变形”的核心：`object` 是 `THREE.Bone`，修改它的 quaternion 后，Three.js 的 skinned mesh 会按骨骼矩阵驱动蒙皮变形。

拖动期间还会派发 `change` 和 `objectChange` 事件：

参考：`src/js/shared/IK/utils/TransformControls.js:570`、`src/js/shared/IK/utils/TransformControls.js:571`。

```js
this.dispatchEvent(changeEvent)
this.dispatchEvent(objectChangeEvent)
```

### 5. pointerup 触发保存

释放鼠标时，`TransformControls.pointerUp` 派发 `pointerup`，外层事件处理器再派发 `transformMouseUp`：

参考：`src/js/shared/IK/utils/TransformControls.js:575`、`src/js/shared/IK/utils/TransformControls.js:581`、`src/js/shared/IK/utils/TransformControls.js:582`、`src/js/shared/IK/utils/TransformControls.js:671`。

```js
mouseUpEvent.mode = this.mode
this.dispatchEvent(mouseUpEvent)
scope.dispatchEvent({ type: "transformMouseUp", value: event })
```

`ObjectRotationControl` 监听 `transformMouseUp` 后调用 `updateCharacter`，也就是 `Character` 注入的 `updateCharacterSkeleton`：

参考：`src/js/shared/IK/objects/ObjectRotationControl.js:68`、`src/js/shared/IK/objects/ObjectRotationControl.js:69`、`src/js/shared/IK/objects/ObjectRotationControl.js:35`、`src/js/shared/IK/objects/ObjectRotationControl.js:37`。

```js
this.control.addEventListener("transformMouseUp", this.onMouseUp, false)

onMouseUp = event => {
  this.updateCharacter && this.updateCharacter(this.object.name, this.object.rotation)
}
```

## Redux 如何保存旋转后的骨骼姿态

`Character` 注入的回调会调用 `updateCharacterSkeleton({ id, name, rotation })`：

参考：`src/js/shot-generator/components/Three/Character.js:411`、`src/js/shot-generator/components/Three/Character.js:412`、`src/js/shot-generator/components/Three/Character.js:413`、`src/js/shot-generator/components/Three/Character.js:414`、`src/js/shot-generator/components/Three/Character.js:415`。

```js
boneRotationControl.current.setUpdateCharacter((name, rotation) => {
  updateCharacterSkeleton({
    id: sceneObject.id,
    name: name,
    rotation: { x: rotation.x, y: rotation.y, z: rotation.z }
  })
})
```

action creator：

参考：`src/js/shared/reducers/shot-generator.js:1762`、`src/js/shared/reducers/shot-generator.js:1763`、`src/js/shared/reducers/shot-generator.js:1764`。

```js
updateCharacterSkeleton: ({ id, name, rotation }) => ({
  type: 'UPDATE_CHARACTER_SKELETON',
  payload: { id, name, rotation }
})
```

reducer 会把 rotation 写到对应人物对象的 `skeleton[boneName].rotation`：

参考：`src/js/shared/reducers/shot-generator.js:1139`、`src/js/shared/reducers/shot-generator.js:1140`、`src/js/shared/reducers/shot-generator.js:1142`、`src/js/shared/reducers/shot-generator.js:1147`。

```js
case 'UPDATE_CHARACTER_SKELETON':
  draft[action.payload.id].skeleton = draft[action.payload.id].skeleton || {}
  if (draft[action.payload.id].skeleton[action.payload.name]) {
    draft[action.payload.id].skeleton[action.payload.name].rotation = { x: rotation.x, y: rotation.y, z: rotation.z }
  } else {
    draft[action.payload.id].skeleton[action.payload.name] = { rotation: action.payload.rotation }
  }
```

后续重新渲染 `Character` 时，`Character` 会把 `sceneObject.skeleton` 中保存的 rotation 应用回 Three.js 骨骼：

参考：`src/js/shot-generator/components/Three/Character.js:145`、`src/js/shot-generator/components/Three/Character.js:151`、`src/js/shot-generator/components/Three/Character.js:154`、`src/js/shot-generator/components/Three/Character.js:156`、`src/js/shot-generator/components/Three/Character.js:165`。

```js
let modified = sceneObject.skeleton[bone.name]
let state = modified || original
if (bone.rotation.equals(state.rotation) == false) {
  bone.rotation.setFromVector3(state.rotation)
  bone.updateMatrixWorld()
}
```

## 圆环为什么能跟随骨骼位置、保持合适大小

`TransformControlsGizmo.updateMatrixWorld` 每帧会把每个 handle 的位置放到 `this.worldPosition`，也就是当前被 attach 对象的世界位置：

参考：`src/js/shared/IK/utils/TransformControls.js:1112`、`src/js/shared/IK/utils/TransformControls.js:1156`、`src/js/shared/IK/utils/TransformControls.js:1158`。

```js
this.updateMatrixWorld = function () {
  handle.visible = true
  handle.position.copy(this.worldPosition)
}
```

随后根据相机距离和 `this.size` 计算缩放，并限制在 `minimumScale = 0.1` 到 `maximumScale = 0.2` 之间：

参考：`src/js/shared/IK/utils/TransformControls.js:740`、`src/js/shared/IK/utils/TransformControls.js:741`、`src/js/shared/IK/utils/TransformControls.js:1162`、`src/js/shared/IK/utils/TransformControls.js:1163`、`src/js/shared/IK/utils/TransformControls.js:1164`、`src/js/shared/IK/utils/TransformControls.js:1168`。

```js
let minimumScale = new THREE.Vector3(0.1, 0.1, 0.1)
let maximumScale = new THREE.Vector3(0.2, 0.2, 0.2)
var eyeDistance = this.worldPosition.distanceTo(this.cameraPosition)
handle.scale.set(1, 1, 1).multiplyScalar(eyeDistance * this.size / 7)
```

这就是截图中控制器不会随着相机距离无限变大或变小的原因。

## 圆环为什么会朝向相机调整

在 rotate 模式下，`TransformControlsGizmo.updateMatrixWorld` 会根据当前坐标空间 quaternion 和相机方向 `eye` 调整 X/Y/Z 圆环的 quaternion：

参考：`src/js/shared/IK/utils/TransformControls.js:1385`、`src/js/shared/IK/utils/TransformControls.js:1389`、`src/js/shared/IK/utils/TransformControls.js:1398`、`src/js/shared/IK/utils/TransformControls.js:1406`、`src/js/shared/IK/utils/TransformControls.js:1414`。

```js
} else if (this.mode === 'rotate') {
  tempQuaternion2.copy(quaternion)
  alignVector.copy(this.eye).applyQuaternion(tempQuaternion.copy(quaternion).inverse())
  if (handle.name === 'X') { ... }
  if (handle.name === 'Y') { ... }
  if (handle.name === 'Z') { ... }
}
```

具体到三个轴：

参考：`src/js/shared/IK/utils/TransformControls.js:1400`、`src/js/shared/IK/utils/TransformControls.js:1402`、`src/js/shared/IK/utils/TransformControls.js:1408`、`src/js/shared/IK/utils/TransformControls.js:1410`、`src/js/shared/IK/utils/TransformControls.js:1416`、`src/js/shared/IK/utils/TransformControls.js:1418`。

```js
tempQuaternion.setFromAxisAngle(unitX, Math.atan2(-alignVector.y, alignVector.z))
handle.quaternion.copy(tempQuaternion)

tempQuaternion.setFromAxisAngle(unitY, Math.atan2(alignVector.x, alignVector.z))
handle.quaternion.copy(tempQuaternion)

tempQuaternion.setFromAxisAngle(unitZ, Math.atan2(alignVector.y, alignVector.x))
handle.quaternion.copy(tempQuaternion)
```

这个逻辑让圆环在人物转动、相机角度变化时仍然保持可读、可操作。

## 命中平面：拖动计算依赖隐藏平面

旋转拖拽不是直接用屏幕像素差，而是射线与一个隐藏巨大平面相交。该平面在 `TransformControlsPlane` 中创建：

参考：`src/js/shared/IK/utils/TransformControls.js:1481`、`src/js/shared/IK/utils/TransformControls.js:1485`、`src/js/shared/IK/utils/TransformControls.js:1486`、`src/js/shared/IK/utils/TransformControls.js:1487`。

```js
THREE.Mesh.call(this,
  new THREE.PlaneBufferGeometry(100000, 100000, 2, 2),
  new THREE.MeshBasicMaterial({ visible: false, side: THREE.DoubleSide })
)
```

rotate 模式下，平面始终平行于相机：

参考：`src/js/shared/IK/utils/TransformControls.js:1550`、`src/js/shared/IK/utils/TransformControls.js:1552`、`src/js/shared/IK/utils/TransformControls.js:1558`、`src/js/shared/IK/utils/TransformControls.js:1559`。

```js
case 'rotate':
  dirVector.set(0, 0, 0)

if (dirVector.length() === 0) {
  this.quaternion.copy(this.cameraQuaternion)
}
```

所以旋转角度来自“鼠标射线在相机平行平面上的点位变化”，再投影到当前轴的旋转方向上。

## 与截图对应的运行时对象关系

截图中红框的三轴紫色圆环，对应运行时层级大致是：

```text
scene.children[0]
└─ ObjectRotationControl.offsetObject              userData.type = "controlTarget"
   └─ TransformControls                            userData.type = "objectControl"
      ├─ TransformControlsGizmo
      │  ├─ gizmo.rotate.X                         可见紫色 X 环
      │  ├─ gizmo.rotate.Y                         可见紫色 Y 环
      │  ├─ gizmo.rotate.Z                         可见紫色 Z 环
      │  └─ picker.rotate.X/Y/Z                    隐形命中环
      └─ TransformControlsPlane                    隐形拖拽计算平面

TransformControls.object -> 当前选中的 THREE.Bone
TransformControls.characterId -> 当前 Character group uuid
ObjectRotationControl.object -> 当前选中的 THREE.Bone
```

关键绑定发生在：

- `Character` 找到骨骼：`src/js/shot-generator/components/Three/Character.js:342`
- `Character` 绑定控制器：`src/js/shot-generator/components/Three/Character.js:346`
- `ObjectRotationControl` attach 目标：`src/js/shared/IK/objects/ObjectRotationControl.js:66`
- `TransformControls` 保存目标对象：`src/js/shared/IK/utils/TransformControls.js:212`、`src/js/shared/IK/utils/TransformControls.js:214`

## 需要特别注意的实现细节

1. **骨骼旋转控制器和人物整体旋转控制器不是同一个实例**：人物整体旋转的 `objectRotationControl` 来自 `SceneManagerR3fLarge`，骨骼旋转的 `boneRotationControl` 在 `Character` 内部创建。
2. **骨骼控制器默认显示三轴**：`ObjectRotationControl` 没有传 axis 参数时，`TransformControls` 默认显示 X/Y/Z；人物整体旋转才通过 `setShownAxis(axis.Y_axis)` 限制成单轴。
3. **可见圆环与命中圆环分离**：可见的是 `gizmo.rotate.X/Y/Z`，鼠标命中检测用的是隐藏的 `picker.rotate.X/Y/Z`。
4. **旋转即时修改 Three.js 骨骼，释放后才持久化到 Redux**：拖动过程中直接改 `THREE.Bone.quaternion`；`transformMouseUp` 时才调用 `updateCharacterSkeleton`。
5. **保存的是欧拉角 rotation，不是 quaternion**：`ObjectRotationControl` 传出 `this.object.rotation`，reducer 保存 `{ x, y, z }`，重新渲染时 `bone.rotation.setFromVector3(...)`。
6. **材质关闭 depthTest/depthWrite**：圆环会稳定浮在人物前方显示，截图中手臂附近的紫色环不会被身体完全遮挡。
