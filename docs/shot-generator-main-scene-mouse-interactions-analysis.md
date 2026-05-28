# Shot Generator 右侧主场景鼠标交互实现分析

本文分析 Shot Generator 右侧 3D 主场景中的鼠标交互：点击选择、Shift 多选、拖拽物体/人物/配件、右键为什么也能拖动人物、空白区域如何控制相机、滚轮和中键/右键的相机控制，以及骨骼/IK/TransformControl 的鼠标命中逻辑。

## 总体结论

右侧主场景鼠标交互主要由两层系统共同完成：

1. **InteractionManager**：负责场景对象命中检测、选择、多选、拖拽物体/人物/配件、骨骼和 IK 控制点选择。
2. **CameraControls**：负责在没有命中可操作对象时的相机控制，包括鼠标拖拽旋转/平移/Orbit/Dolly、滚轮调整 FOV。

你观察到的“鼠标右键选中人物能移动人物位置”，根因是：

- `InteractionManager.onPointerDown` 没有区分左键/右键；只要命中的是当前已选中的对象，就会 `setDragTarget(...)`。
- `useDraggingManager.drag(...)` 也不判断 `event.button`，只按 `dragTarget` 移动物体。
- 因此在人物已经被选中的情况下，无论左键还是右键按在人身上并拖动，都会进入对象拖拽逻辑，而不是相机右键 Orbit。
- 只有点在空白处，或者没有进入对象拖拽时，右键才会被传给 `CameraControls` 做相机 Orbit。

## 关键代码位置

- 主场景交互管理：`src/js/shot-generator/components/Three/InteractionManager.js`
- 拖拽实现：`src/js/shot-generator/hooks/use-dragging-manager.js`
- 相机鼠标控制：`src/js/shot-generator/CameraControls.js`
- 相机控制组件桥接：`src/js/shot-generator/components/Three/CameraControlsComponet.js`
- 主场景挂载入口：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 骨骼 Helper：`src/js/xr/src/three/BonesHelper.js`
- IK Helper：`src/js/shared/IK/SGIkHelper.js`
- TransformControls：`src/js/shared/IK/utils/TransformControls.js`

## 事件监听入口

`InteractionManager` 会直接把 pointer 事件绑定到右侧主场景的 `activeGL.domElement` 上，并把 `pointerup` 绑定到 window，避免拖出 canvas 后丢失释放事件。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:437`。

```js
activeGL.domElement.addEventListener('pointerdown', onPointerDown)
activeGL.domElement.addEventListener('pointermove', onPointerMove)
activeGL.domElement.addEventListener('pointermove', throttleUpdateDraggableObject)
window.addEventListener('pointerup', onPointerUp)
```

相机控制由 `CameraControlsComponent` 桥接，`InteractionManager` 把没有被对象交互消费的 pointerDown / pointerUp 事件传给它：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:450`。

```jsx
<CameraControlsComponent
  pointerDownEvent={pointerDownEvent}
  pointerUpEvent={pointerUpEvent}
  activeGL={activeGL}
  isCameraControlsEnabled={isCameraControlsEnabled}
  takeSceneObjects={takeSceneObjects}
  activeCamera={activeCamera}
/>
```

`CameraControlsComponent` 收到事件后调用 `CameraControls.onPointerDown/onPointerUp`：

参考：`src/js/shot-generator/components/Three/CameraControlsComponet.js:120`。

```js
useEffect(() => {
  if(!pointerDownEvent) return
  let sceneObjects = takeSceneObjects()
  cameraControlsView.current.object = CameraControls.objectFromCameraState(sceneObjects[activeCamera])
  cameraControlsView.current.onPointerDown(pointerDownEvent)
}, [pointerDownEvent])
```

## 命中检测：鼠标点到的是谁

### 1. 坐标转换

`InteractionManager.mouse(event)` 将浏览器坐标转换为 Three.js 标准 NDC 坐标：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:144`。

```js
const mouse = event => {
  const rect = activeGL.domElement.getBoundingClientRect()
  return {
    x: ((event.clientX - rect.left) / rect.width) * 2 - 1,
    y: -((event.clientY - rect.top) / rect.height) * 2 + 1
  }
}
```

### 2. 可命中对象列表

每次 pointerDown 会重新整理可命中对象：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:135`。

```js
const filterIntersectables = () => {
  intersectables.current = scene.__interaction
  intersectables.current = intersectables.current.concat(scene.children[0].children.filter(o =>
    o.userData.type === 'controlTarget' ||
    o.userData.type === 'controlPoint' ||
    o.userData.type === 'objectControl' ||
    o.userData.type === 'group'))
}
```

其中 `scene.__interaction` 通常包含人物、物体、灯光、图片、配件等可交互对象；另外再拼上控制点、TransformControl、group 等辅助对象。

### 3. 命中优先级

`getIntersects(pointer)` 先检测 IK Helper，再使用 GPU Picker 选中场景对象：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:152`。

```js
raycaster.current.setFromCamera({ x, y }, camera)
let intersects = raycaster.current.intersectObject(SGIkHelper.getInstance())
if(intersects.length > 0) {
  return intersects
}

gpuPicker.setupScene((intersectables.current || []).filter(object => object.userData.type !== 'volume'))
gpuPicker.controller.setPickingPosition(mousePosition.current.x, mousePosition.current.y)
intersects = gpuPicker.pickWithCamera(camera, activeGL)
```

也就是说：

1. IK 控制点/辅助器优先。
2. 然后才是普通场景对象。
3. `volume` 被排除在 GPU Picker 外。

### 4. 命中对象归一化

不同 mesh / helper 命中后会被归一化成可操作的目标对象：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:30`。

```js
const getIntersectionTarget = intersect => {
  if (intersect.object.userData.type === 'hitter') {
    return intersect.object.parent.object3D
  }

  if(intersect.object.type === 'gizmo') {
    if(intersect.object.parent.parent.userData.type === 'objectControl') {
      return intersect.object.parent.parent.parent
    }
    return intersect.object
  }

  if(intersect.object.userData.type === 'controlPoint' || intersect.object.userData.type === 'objectControl'
    || intersect.object.userData.type === 'poleTarget') {
    return intersect.object
  }

  if(intersect.object.type === 'SkinnedMesh') {
    return intersect.object.parent.parent
  }

  if(intersect.object.parent.userData.type === 'object' || intersect.object.userData.type === 'attachable'
    || intersect.object.parent.userData.type === 'image' || intersect.object.userData.type === 'light') {
    return intersect.object.parent
  }
}
```

这段逻辑解释了为什么点到人物 mesh、物体 mesh、灯光 mesh、配件 mesh，最终都能回到对应的 scene object group。

## 点击空白区域：切回 active camera 并进入相机控制

当 pointerDown 没有命中任何对象时：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:190`。

```js
if (intersects.length === 0) {
  if(dragTarget || (selections[0] !== activeCamera) ) {
    endDrag()
    setDragTarget(null)
    setLastDownId(null)
    selectObject(activeCamera)
    selectBone(null)
  }
  setOnPointDown(event)
}
```

行为：

- 如果当前选中的不是 active camera，就切回 active camera。
- 清空选中的骨骼。
- 把 pointerDown 事件传给相机控制。

pointerUp 时也会传给相机控制：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:387`。

```js
enableCameraControls(true)
setOnPointUp(event)
```

## 点击对象：选择、Shift 多选、选择 group

### 1. 普通点击选择

在 pointerDown 时，如果对象不是当前 selection 里的对象，先只记录 `lastDownId`，不会立刻选择；真正选择发生在 pointerUp，且要求 pointerUp 命中的是同一个对象。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:321` 和 `src/js/shot-generator/components/Three/InteractionManager.js:407`。

```js
setLastDownId(target.userData.id)
```

pointerUp：

```js
if (target && target.userData.id == lastDownId) {
  if (!selections.includes(target.userData.id) && !target.userData.locked && !target.userData.blocked) {
    let object = sceneObjects[target.userData.id]
    if (object && object.group) {
      selectObject([object.group, ...sceneObjects[object.group].children])
    } else {
      selectObject(target.userData.id)
    }
  }
}
```

这能避免拖拽时误选，也支持点击 group 子对象时选中整个 group。

### 2. Shift 多选

pointerUp 时如果按着 Shift：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:408`。

```js
if (event.shiftKey) {
  if (selections.length === 1 && selections[0] === activeCamera) {
    selectObject(target.userData.id)
  } else {
    selectObjectToggle(target.userData.id)
  }
}
```

行为：

- 如果当前只选中了 active camera，则 Shift 点击对象会替换选择。
- 否则 Shift 点击会 toggle 当前对象的选择状态。

## 拖拽物体/人物：为什么右键也能移动人物

### 1. 进入拖拽条件

当当前已有 selection，并且 pointerDown 命中的是已选对象时：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:313`。

```js
if (selections.includes(target.userData.id)) {
  shouldDrag = true
  blockObject(target.userData.id)
}
```

随后：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:323`。

```js
if (shouldDrag) {
  setDragTarget({ target, x, y, isObjectControl: target.isRotated })
} else {
  setOnPointDown(event)
}
```

这里没有判断 `event.button`，所以左键、右键、中键只要点在已选对象上，都可以进入拖拽。

### 2. 初始化拖拽

`dragTarget` 一旦设置，`useMemo` 会触发拖拽准备，并禁用相机控制：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:171`。

```js
if(dragTarget && dragTarget.target){
  let selections = takeSelections()
  let { target, x, y } = dragTarget
  enableCameraControls(false)
  prepareDrag(target, { x, y, useIcons:true, camera, scene, selections })
  undoGroupStart()
}
```

所以右键点在已选人物上时，会优先变成对象拖拽，不会进入相机右键 Orbit。

### 3. 拖拽更新

pointerMove 时，如果有 dragTarget 且不是 objectControl，调用 `drag(...)`：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:337`。

```js
if (dragTarget && !dragTarget.isObjectControl) {
  if(dragTarget.target.userData.type === 'character') {
    let ikRig = SGIkHelper.getInstance().ragDoll
    if(!ikRig || !ikRig.isEnabledIk && !ikRig.hipsMoving && !ikRig.hipsMouseDown) {
      drag({ x, y }, dragTarget.target, camera, selections, event.ctrlKey)
    }
  } else {
    drag({ x, y }, dragTarget.target, camera, selections, event.ctrlKey)
  }
}
```

`event.ctrlKey` 会作为 `isRotation` 传给 drag，用于单人物旋转。

### 4. 拖拽算法

`useDraggingManager.prepareDrag` 会以目标位置创建一个与相机方向垂直的平面，并记录鼠标点与对象位置的 offset。

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:13`。

```js
plane.current.setFromNormalAndCoplanarPoint(camera.getWorldDirection(plane.current.normal), target.position)
```

普通对象/人物拖拽时只改水平面位置：

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:83`。

```js
let { x, z } = intersection.current.clone().sub(offsets.current[selection]).setY(0)
target.position.set(x, target.position.y, z)
objectChanges.current[selection] = { x, y: z }
```

注意这里把 Three 世界的 `z` 存成数据层的 `y`，因为 Shot Generator 数据层和 Three 坐标轴有转换。

### 5. Ctrl + 拖拽人物：旋转人物

如果拖拽时按住 Ctrl，且目标是单个人物，会改人物 `rotation.y`，而不是改位置：

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:84`。

```js
if(isRotation && target.userData.type === 'character' && selections.length === 1) {
  let delta = prevMouse.current.x - mouse.x
  let rotationSpeed = Math.sign(delta) * (THREE.Math.DEG2RAD * (Math.abs(delta) * 180))
  let newRotation = target.rotation.y + ((rotationSpeed) * direction)
  objectChanges.current[selection] = { rotation: newRotation }
  target.rotation.y = newRotation
}
```

### 6. 拖拽保存

拖拽过程中每 16ms 节流写入 store：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:333`。

```js
const throttleUpdateDraggableObject = throttle(() => {
  updateStore(updateObjects)
}, 16, {trailing: true})
```

但如果是 Ctrl 旋转，`updateStore` 会跳过中途写入，只在 pointerUp 的 `endDrag(updateObjects)` 写入最终状态：

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:104`。

```js
if (!objectChanges.current || !Object.keys(objectChanges.current).length || isCtrlPressed.current) {
  return false
}
updateObjects(objectChanges.current)
```

pointerUp：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:355`。

```js
if (dragTarget && dragTarget.target) {
  endDrag(updateObjects)
  unblockObject(dragTarget.target.userData.id)
  setDragTarget(null)
  undoGroupEnd()
}
```

## 拖拽人物后同步配件

人物拖拽结束时，如果目标是 character，会重新计算所有绑定到该人物的 attachable 世界位置/旋转，并写回 Redux。

参考：`src/js/shot-generator/components/Three/InteractionManager.js:362`。

```js
if(dragTarget.target.userData.type === 'character') {
  let attachables = scene.__interaction.filter(object => object.userData.bindedId === dragTarget.target.userData.id)
  for(let i = 0; i < attachables.length; i ++) {
    let attachable = attachables[i]
    attachable.parent.updateWorldMatrix(true, true)
    let position = attachable.worldPosition()
    let quaternion = attachable.worldQuaternion()
    let matrix = attachable.matrix.clone()
    matrix.premultiply(attachable.parent.matrixWorld)
    matrix.decompose(position, quaternion, new THREE.Vector3())
    let rot = new THREE.Euler().setFromQuaternion(quaternion, 'XYZ')
    dispatch(updateObject(attachable.userData.id, {
      x: position.x,
      y: position.y,
      z: position.z,
      rotation: { x: rot.x, y: rot.y, z: rot.z }
    }))
  }
}
```

这保证人物移动后，头发、眼镜、面具、道具等附着物的 store 状态也同步更新。

## 拖拽配件

如果 pointerDown 命中 `userData.type === 'attachable'`：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:225`。

```js
if(target.userData && target.userData.type === 'attachable') {
  selectAttachable({ id: target.userData.id, bindId: target.userData.bindedId })
  setDragTarget({ target, x, y})
  return
}
```

行为：

- 选中配件。
- selection 仍保持绑定人物 id，因为 `selectAttachable` 会用 `bindId` 设置人物 selection。
- 进入配件拖拽。

配件拖拽不同于普通对象，它允许在 3D 空间里改 `x/y/z`，且需要在骨骼局部空间和世界空间之间转换。

参考：`src/js/shot-generator/hooks/use-dragging-manager.js:65`。

```js
if(target.userData.type === 'attachable') {
  if(target.userData.isRotationEnabled) return
  let { x, y, z } = intersection.current.clone().sub(offsets.current[target.userData.id])
  let parentMatrixWorld = target.parent.matrixWorld
  let parentInverseMatrixWorld = target.parent.getInverseMatrixWorld()
  target.applyMatrix4(parentMatrixWorld)
  target.position.set(x, y, z)
  target.updateMatrixWorld(true)
  target.applyMatrix4(parentInverseMatrixWorld)
  target.updateMatrixWorld(true)

  objectChanges.current[target.userData.id] = { x, y, z }
}
```

如果配件当前处于旋转模式，拖拽移动会被跳过：

```js
if(target.userData.isRotationEnabled) return
```

## 点击人物骨骼 / IK 控制点

当当前只选中一个人物，且 pointerDown 命中的是同一个人物时，会额外检测骨骼 helper：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:298`。

```js
raycaster.current.setFromCamera({ x, y }, camera)
let hits = raycaster.current.intersectObject(BonesHelper.getInstance())
if (!isSelectedControlPoint && hits.length) {
  selectObject(target.userData.id)
  blockObject(target.userData.id)
  setLastDownId(null)
  selectBone(hits[0].bone.uuid)
  setDragTarget({ target, x, y })
  return
}
```

行为：

- 点到骨骼时会 `selectBone(...)`。
- 同时设置 dragTarget，后续骨骼旋转/控制逻辑由 BonesHelper / TransformControls / IK 系统处理。

如果命中的是 IK controlPoint / poleTarget：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:229`。

```js
SGIkHelper.getInstance().selectControlPoint(target.uuid, event)
let characters = intersectables.current.filter(value => value.uuid === characterId)
target = characters[0]
isSelectedControlPoint = true
```

命中 objectControl / gizmo 时，则会转交给 object rotation/transform control，不走普通拖拽：

参考：`src/js/shot-generator/components/Three/InteractionManager.js:235`。

```js
if(target.userData && target.userData.type === 'objectControl') {
  let targetElement = target.object
  ...
  setDragTarget({ target, x, y, isObjectControl: true })
  return
}
```

`isObjectControl: true` 会让 `InteractionManager.onPointerMove` 不调用普通 `drag(...)`。

## 相机鼠标控制

当点击空白区域或非对象交互时，事件会传给 `CameraControls`。

### 1. pointerDown / pointerUp 状态

参考：`src/js/shot-generator/CameraControls.js:100`。

```js
onPointerDown(event) {
  event.preventDefault()
  event.stopPropagation()
  this.domElement.focus()
  ...
  this.mouseDragOn = true
  if(event.button === 2) {
    this.isRightButtonPressed = true
  }
  if(event.button === 1) {
    this.isMiddleButtonPressed = true
  }
  this.onChange({active: this.mouseDragOn, object: this._object})
}
```

pointerUp 会保存相机状态并结束 undo：

参考：`src/js/shot-generator/CameraControls.js:139`。

```js
if (this.mouseDragOn && this.enabled === true ) {
  this.onChange({active: false, object: this._object})
  this.undoGroupEnd()
}
this.mouseDragOn = false
this.isRightButtonPressed = false
this.isMiddleButtonPressed = false
```

### 2. 左键拖拽空白：相机 Pan/Tilt

没有任何修饰键时，空白区域拖拽会修改 active camera 的 `rotation` 和 `tilt`：

参考：`src/js/shot-generator/CameraControls.js:506`。

```js
let rotation = (this.mouseX - this.prevMouseX)*0.001
this._object.rotation -= rotation
let tilt = (this.mouseY - this.prevMouseY)*0.001
this._object.tilt -= tilt
this._object.tilt = Math.max(Math.min(this._object.tilt, Math.PI / 2), -Math.PI / 2)
```

### 3. 右键拖拽空白 / Ctrl + 拖拽：相机 Orbit

右键或 Ctrl 时进入 Orbit：

参考：`src/js/shot-generator/CameraControls.js:378`。

```js
else if(this.controlPressed || this.isRightButtonPressed) {
  ...
  spherical.theta += rotation
  spherical.phi += tilt
  spherical.makeSafe()
  camera.position.addVectors(target, offset)
  camera.lookAt(target)
  ...
}
```

如果当前选中了非相机对象，`CameraControlsComponent` 会把相机 orbit target 设置为选中对象中心；如果选中的是人物，内置人物会用 Head 骨骼位置作为目标。

参考：`src/js/shot-generator/components/Three/CameraControlsComponet.js:33`。

```js
if(selectedObject.userData.type === 'character') {
  if(!isUserModel(selectedObject.userData.model)) {
    let skinnedMesh = selectedObject.getObjectByProperty('type', 'SkinnedMesh')
    let bone = skinnedMesh.skeleton.getBoneByName('Head')
    target.add(bone.worldPosition())
  }
}
```

### 4. 中键拖拽空白 / Alt + 拖拽：相机平移

中键或 Alt 时，进入 truck / pedestal：

参考：`src/js/shot-generator/CameraControls.js:480`。

```js
else if(this.altPressed || this.isMiddleButtonPressed ) {
  let horizontalDelta = (this.mouseX - this.prevMouseX)*0.005
  ...
  camera.position.add(cameraHorizontalDirection)

  let verticalDelta = (this.mouseY - this.prevMouseY)*0.005
  ...
  camera.position.sub(cameraVerticalDirection)

  this._object.x = position.x
  this._object.y = position.z
  this._object.z = position.y
}
```

### 5. Shift + 拖拽空白：相机 Dolly

Shift 时，沿相机视线方向移动：

参考：`src/js/shot-generator/CameraControls.js:463`。

```js
else if(this.shiftPressed) {
  let verticalDelta = (this.mouseY - this.prevMouseY)*0.015
  camera.getWorldDirection(cameraVerticalDirection)
  cameraVerticalDirection.normalize()
  cameraVerticalDirection.multiplyScalar(verticalDelta)
  camera.position.sub(cameraVerticalDirection)
}
```

### 6. Shift + Alt / Shift + 中键：Dolly Zoom

Shift + Alt 或 Shift + 中键时，同时调整相机位置和 FOV，保持画面目标大小：

参考：`src/js/shot-generator/CameraControls.js:333`。

```js
if(this.shiftPressed && (this.altPressed || this.isMiddleButtonPressed )) {
  let verticalDelta = (this.mouseY - this.prevMouseY)*0.010
  ...
  let fov = 2 * Math.atan(this.initialHeight / (2 * distance)) * THREE.Math.RAD2DEG
  this._object.x = position.x
  this._object.y = position.z
  this._object.z = position.y
  this._object.fov = fov
}
```

### 7. 滚轮：调整 FOV

滚轮不走 InteractionManager，直接在 CameraControls 里监听：

参考：`src/js/shot-generator/CameraControls.js:260`。

```js
onWheel(event) {
  this.zoomSpeed += (event.deltaY * 0.005)
}
```

随后在 `update` 中应用到 FOV，并限制范围：

```js
this._object.fov += this.zoomSpeed
this._object.fov = Math.max(1, this._object.fov)
this._object.fov = Math.min(90, this._object.fov)
this.zoomSpeed = this.zoomSpeed * 0.0001
```

## 鼠标功能清单

| 操作 | 命中位置 | 功能 | 主要代码 |
|---|---|---|---|
| 单击空白 | 空白区域 | 选择 active camera，进入相机控制 | `InteractionManager.js:190` |
| 左键拖拽空白 | 空白区域 | 相机 pan / tilt | `CameraControls.js:506` |
| 右键拖拽空白 | 空白区域 | 相机 orbit | `CameraControls.js:378` |
| 中键拖拽空白 | 空白区域 | 相机 truck / pedestal | `CameraControls.js:480` |
| Shift + 拖拽空白 | 空白区域 | 相机 dolly | `CameraControls.js:463` |
| Shift + Alt / Shift + 中键拖拽空白 | 空白区域 | Dolly zoom | `CameraControls.js:333` |
| 滚轮 | 主场景 | 调整 active camera FOV | `CameraControls.js:260` |
| 单击对象 | object / character / light / image | 选中对象 | `InteractionManager.js:407` |
| Shift + 单击对象 | object / character / light / image | 多选 toggle | `InteractionManager.js:408` |
| 拖拽已选对象 | 已选 object/character/image/light 等 | 移动对象 | `InteractionManager.js:313`、`use-dragging-manager.js:83` |
| 右键拖拽已选人物 | 已选 character | 移动人物位置 | 因为 `setDragTarget` 不判断 button |
| Ctrl + 拖拽单个人物 | 已选 character | 旋转人物 | `use-dragging-manager.js:84` |
| 点击配件 | attachable | 选中配件并可拖动 | `InteractionManager.js:225` |
| 拖拽配件 | attachable | 移动配件，写回 x/y/z | `use-dragging-manager.js:65` |
| 点击骨骼 | 选中人物的 BonesHelper | 选中骨骼 | `InteractionManager.js:298` |
| 点击 IK 控制点 | controlPoint / poleTarget | 选中 IK 控制点 | `InteractionManager.js:229` |
| 拖拽 gizmo/objectControl | TransformControl | 旋转/平移被控对象/骨骼/配件 | `InteractionManager.js:235` |

## 右键移动人物的具体原因

你的场景里“右键点人物能移动人物位置”的实现链路是：

1. 人物已经在 `selections` 中。
2. 鼠标右键按下，`InteractionManager.onPointerDown` 命中人物。
3. 代码只判断 `selections.includes(target.userData.id)`，不判断 `event.button`。
4. 因此设置：

```js
setDragTarget({ target, x, y, isObjectControl: target.isRotated })
```

5. `dragTarget` 触发 `prepareDrag(...)`，同时 `enableCameraControls(false)`。
6. 鼠标移动时调用 `drag(...)`。
7. `drag(...)` 修改人物 Three 对象位置，并通过 `updateObjects` 写回 Redux。

核心代码：

- `src/js/shot-generator/components/Three/InteractionManager.js:313`
- `src/js/shot-generator/components/Three/InteractionManager.js:323`
- `src/js/shot-generator/components/Three/InteractionManager.js:337`
- `src/js/shot-generator/hooks/use-dragging-manager.js:83`

如果未来希望“右键在人物上也始终 Orbit 相机”，需要在 `InteractionManager.onPointerDown` 中对 `event.button === 2` 做特殊处理，不进入 `setDragTarget`，而是走 `setOnPointDown(event)` 传给 `CameraControls`。

## 注意点

- `InteractionManager` 没有显式区分左键/右键进行对象拖拽，这是右键拖动已选对象的根本原因。
- 主场景没有看到单独的 `contextmenu` 禁用逻辑；右键行为主要依赖 pointer event 的 `button === 2`。
- 对象拖拽时会禁用相机控制，避免相机和对象同时动。
- 空白区域拖拽才会进入相机控制；命中已选对象会优先进入对象拖拽。
- 配件、人物、普通对象、骨骼、IK 控制点、TransformControl 都走不同分支，排查鼠标问题时优先看 `getIntersectionTarget` 和 `onPointerDown` 的分支。
