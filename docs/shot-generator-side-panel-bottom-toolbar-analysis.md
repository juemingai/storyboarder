# Shot Generator 左侧功能区与底部工具栏功能/实现说明

本文档说明 Shot Generator 页面中：

- 左上角相机俯视预览框下方的左侧功能区
- 右侧主场景下方的底部相机工具栏

每个功能的作用、数据流和实现代码位置，供后续移植到 FlowCanvas 3D 导演台时参考。

## 1. 页面布局总览

整体布局入口在：

- `src/js/shot-generator/components/Editor/index.js`

核心结构：

```jsx
<div id="sg-main">
  <div id="aside">
    <div id="topdown">...</div>
    <div id="elements">
      <ElementsPanel/>
    </div>
  </div>

  <div className="column fill">
    <div id="camera-view">...</div>
    <div className="inspectors">
      <CameraPanelInspector/>
      <BoardInspector/>
      <div style={{ flex: "1 1 auto" }}>
        <CamerasInspector/>
        <GuidesInspector/>
      </div>
    </div>
  </div>
</div>
```

对应代码：

- 左侧俯视预览框：`src/js/shot-generator/components/Editor/index.js`
- 左侧功能区：`src/js/shot-generator/components/ElementsPanel/index.js`
- 底部相机工具栏：`src/js/shot-generator/components/CameraPanelInspector/index.js`
- 底部 Shot 信息：`src/js/shot-generator/components/BoardInspector/index.js`
- 底部相机切换：`src/js/shot-generator/components/CamerasInspector/index.js`
- 底部导览线开关：`src/js/shot-generator/components/GuidesInspector/index.js`

## 2. 左侧功能区：ElementsPanel

左侧功能区位于相机俯视预览框下方，主要包含两块：

```text
ElementsPanel
├── ItemList       场景对象列表
└── Inspector      当前选中对象/场景属性面板
```

入口文件：

- `src/js/shot-generator/components/ElementsPanel/index.js`

核心代码：

```jsx
return (
  <div style={{ flex: 1, display: "flex", flexDirection: "column" }}>
    <div id="listing">
      <ItemList/>
    </div>
    <Inspector
      {...{
        world,
        kind,
        data,
        models,
        updateObject,
        selectedBone,
        selectBone,
        updateCharacterSkeleton,
        updateWorld,
        updateWorldRoom,
        updateWorldEnvironment,
        updateWorldFog,
        storyboarderFilePath,
        selections
      }}
    />
  </div>
)
```

### 2.1 ElementsPanel 数据来源

`ElementsPanel` 通过 Redux 读取：

- `world`
- `sceneObjects`
- `selections`
- `selectedBone`
- `models`
- `activeCamera`
- `storyboarderFilePath`

参考代码：

- `src/js/shot-generator/components/ElementsPanel/index.js`

```js
state => ({
  world: getWorld(state),
  sceneObjects: getSceneObjects(state),
  selections: getSelections(state),
  selectedBone: getSelectedBone(state),
  models: state.models,
  activeCamera: getActiveCamera(state),
  storyboarderFilePath: state.meta.storyboarderFilePath
})
```

支持派发的核心 action：

```js
selectObject
selectObjectToggle
updateObject
deleteObjects
setActiveCamera
selectBone
updateCharacterSkeleton
updateWorld
updateWorldRoom
updateWorldEnvironment
updateWorldFog
```

这些 action 统一来自：

- `src/js/shared/reducers/shot-generator.js`

## 3. 左侧对象列表：ItemList

对象列表对应截图中的：

```text
Scene
Camera 1
Camera 2
Adult-male 1
```

实现文件：

- `src/js/shot-generator/components/ItemList/index.js`
- `src/js/shot-generator/components/ItemList/Item.js`

### 3.1 列表排序规则

列表按对象类型排序：

```js
const sortPriority = ['camera', 'character', 'object', 'image', 'light', 'volume', 'group']
```

实现位置：

- `src/js/shot-generator/components/ItemList/index.js`

排序逻辑：

```js
const getSortedItems = (sceneObjectsArray) => {
  const headItems = sceneObjectsArray
    .filter(object => sortPriority.indexOf(object.type) !== -1)
    .sort((prev, current) => sortPriority.indexOf(prev.type) - sortPriority.indexOf(current.type))
    .filter(object => !!object.group === false)

  const sortedItems = []

  for (let object of headItems) {
    sortedItems.push(object)
    if (object.children) {
      sortedItems.push(...sceneObjectsArray.filter(target => target.group === object.id))
    }
  }

  return sortedItems
}
```

作用：

- Camera 始终排在前面。
- Character 在 Camera 后面。
- Object / Image / Light / Volume / Group 依次显示。
- group 的 children 会缩进显示。

### 3.2 点击对象：选择对象

作用：

- 点击列表项选中对应对象。
- 点击 `Scene` 行取消当前对象选择，显示场景属性。
- 点击 Camera 项时，同时切换 active camera。
- `Shift + 点击` 支持多选/切换选择。

实现位置：

- `src/js/shot-generator/components/ItemList/index.js`

核心代码：

```js
const onSelectItem = useCallback((event, props) => {
  if (!props) {
    if (selections.length) {
      selectObject(null)
    }

    return false
  }

  let currentSelections = props.children ? [props.id, ...props.children] : [props.id]
  if (event.shiftKey) {
    if (selections.indexOf(props.id) === -1) {
      currentSelections.push(props.id, ...selections)
    } else {
      currentSelections = selections.filter(id => currentSelections.indexOf(id) === -1)
    }
  }

  selectObject([...new Set(currentSelections)])
  if(props.type === "camera") {
    setActiveCamera(props.id)
  }
}, [selections])
```

数据流：

```text
点击 Item
→ onSelectItem
→ selectObject(id)
→ 如果 type === camera，则 setActiveCamera(id)
→ 左侧 Inspector 显示该对象属性
→ 右侧主场景 active camera / selection 同步变化
```

### 3.3 Active Camera 标识

截图中 Camera 行右侧的小圆点表示当前 active camera。

实现位置：

- `src/js/shot-generator/components/ItemList/Item.js`

```js
const getActiveIcon = (props) => {
  return (
    (props.type === 'camera' && props.activeCamera === props.id)
      ? <span className='active'>
          <Icon src='icon-item-active'/>
        </span>
      : null
  )
}
```

作用：

- 告诉用户当前右侧主视图使用的是哪台相机。
- 点击其他 Camera 行会切换 active camera。

### 3.4 锁定对象

截图中 Camera 2 行右侧的锁图标表示该对象已锁定。

实现位置：

- `src/js/shot-generator/components/ItemList/Item.js`

```js
const getLockIcon = (props) => {
  return (
    <a
      className={classNames({
        'lock': true,
        'hide-unless-hovered': !props.locked
      })}
      onClick={ stopPropagation((e) => props.onLockItem(e, props)) }
    >
      <Icon src={props.locked ? 'icon-item-lock' : 'icon-item-unlock'}/>
    </a>
  )
}
```

状态更新：

- `src/js/shot-generator/components/ItemList/index.js`

```js
const onLockItem = useCallback((event, props) => {
  let nextAvailability = !props.locked
  updateObject(props.id, {locked: nextAvailability})
  if (props.children) {
    for (let child of props.children) {
      updateObject(child, {locked: nextAvailability})
    }
  }
}, [])
```

锁定后的作用：

- 对象不能被拖动或修改。
- `updateObject` 会检查 `locked` 状态。

相关 reducer：

- `src/js/shared/reducers/shot-generator.js`

```js
if (draft.locked || draft.blocked) {
  return
}
```

### 3.5 显示/隐藏对象

作用：

- 非 Camera 对象可以显示/隐藏。
- Camera 不显示 visibility 图标。

实现位置：

- `src/js/shot-generator/components/ItemList/Item.js`

```js
const getVisibilityIcon = (props) => {
  return (
    props.type === 'camera' ? null :
    <a
      className={classNames({
        'visibility': true,
        'hide-unless-hovered': props.visible
      })}
      onClick={ stopPropagation((e) => props.onHideItem(e, props)) }
    >
      <Icon src={props.visible ? 'icon-item-visible' : 'icon-item-hidden'}/>
    </a>
  )
}
```

状态更新：

```js
const onHideItem = useCallback((event, props) => {
  let nextVisibility = !props.visible
  updateObject(props.id, {visible: nextVisibility})
  if (props.children) {
    for (let child of props.children) {
      updateObject(child, {visible: nextVisibility})
    }
  }
}, [])
```

### 3.6 删除对象

作用：

- 删除当前对象。
- active camera 不允许删除。
- 删除 character 时，会同时删除绑定到该角色的 attachable。

实现位置：

- `src/js/shot-generator/components/ItemList/index.js`

是否允许删除：

```js
const allowDelete = props.type !== 'camera' || (props.type === 'camera' && activeCamera !== props.id)
```

删除逻辑：

```js
const onDeleteItem = useCallback((event, props) => {
  dialog.showMessageBox(null, {
    type: 'question',
    buttons: ['Yes', 'No'],
    message: 'Are you sure?',
    defaultId: 1
  })
  .then(({ response }) => {
    if (response === 0) {
      let idsToRemove = props.children ? [...props.children, props.id] : [props.id]
      if(props.type === "character") {
        withState((dispatch, state) => {
          let sceneObjects = getSceneObjects(state)
          let attachableIds = Object.values(sceneObjects)
            .filter(obj => obj.attachToId === props.id)
            .map(obj => obj.id)
          idsToRemove = attachableIds.concat(idsToRemove)
        })
      }
      deleteObjects(idsToRemove)
    }
  })
}, [])
```

## 4. 左侧 Inspector：对象/场景属性区

左侧列表下方是 Inspector 区域。它根据当前选择状态显示不同内容。

入口文件：

- `src/js/shot-generator/components/ElementsPanel/Inspector.js`

核心逻辑：

```jsx
return <div id="inspector">
  {(selectedCount > 1)
    ? <MultiSelectionInspector/>
    : (kind && data)
      ? <InspectedElement/>
      : <InspectedWorld/>}
</div>
```

显示规则：

| 当前状态 | 显示组件 | 说明 |
|---|---|---|
| 未选中对象 / 点 Scene | `InspectedWorld` | 场景/世界属性 |
| 选中单个对象 | `InspectedElement` | 当前对象属性 |
| 多选对象 | `MultiSelectionInspector` | 多选批量属性 |

### 4.1 InspectedElement 标签页

实现文件：

- `src/js/shot-generator/components/InspectedElement/index.js`

它会根据对象类型显示不同标签页：

| 对象类型 | 标签页 |
|---|---|
| camera | 参数 |
| object | 参数、模型 |
| character | 参数、手势、姿势、模型、挂载物、表情、头发 |
| image | 参数 |
| light | 参数 |
| volume | 参数 |

核心代码：

```jsx
<Tabs key={id}>
  <div className="tabs-header">
    <Tab><Icon src="icon-tab-parameters"/></Tab>
    {handPoseTab.tab}
    {charPoseTab.tab}
    {modelTab.tab}
    {attachmentTab.tab}
    {emotionsTab.tab}
    {hairInspectorTab.tab}
  </div>

  <div className="tabs-body">
    <Panel><GeneralInspector/></Panel>
    {handPoseTab.panel}
    {charPoseTab.panel}
    {modelTab.panel}
    {attachmentTab.panel}
    {emotionsTab.panel}
    {hairInspectorTab.panel}
  </div>
</Tabs>
```

### 4.2 Camera 属性面板

截图中选中 Camera 2 后，左侧显示：

```text
X
Y
Z
旋转
滚动
倾斜
F.O.V.
```

实现文件：

- `src/js/shot-generator/components/InspectedElement/GeneralInspector/Camera.js`

字段说明：

| 字段 | 作用 | 状态字段 |
|---|---|---|
| X | 相机水平位置 | `camera.x` |
| Y | 相机地面前后位置 | `camera.y` |
| Z | 相机高度 | `camera.z` |
| 旋转 | 水平旋转 / pan | `camera.rotation` |
| 滚动 | 镜头 roll | `camera.roll` |
| 倾斜 | 上下俯仰 / tilt | `camera.tilt` |
| F.O.V. | 视野角 | `camera.fov` |

核心代码：

```jsx
<NumberSlider label="X" value={props.x} min={-30} max={30} onSetValue={setX} />
<NumberSlider label="Y" value={props.y} min={-30} max={30} onSetValue={setY} />
<NumberSlider label="Z" value={props.z} min={-30} max={30} onSetValue={setZ} />
```

旋转/滚动/倾斜使用角度显示，但写入状态时转换成弧度：

```js
const setRotation = useCallback((value) => updateObject(id, {
  rotation: THREE.Math.degToRad(value)
}), [])

const setRoll = useCallback((value) => updateObject(id, {
  roll: THREE.Math.degToRad(value)
}), [])

const setTilt = useCallback((value) => updateObject(id, {
  tilt: THREE.Math.degToRad(value)
}), [])
```

FOV 更新：

```js
const setFOV = useCallback((fov) => updateObject(id, {fov}), [])
```

数据流：

```text
用户拖动 NumberSlider
→ updateObject(camera.id, patch)
→ Redux sceneObjects 更新
→ CameraUpdate 同步到主 Three.js camera
→ 右侧主场景实时变化
→ 左上角 CameraIcon / 视锥同步变化
```

### 4.3 GeneralInspector 类型分发

不同对象类型对应不同属性面板。

实现文件：

- `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js`

核心代码：

```js
const InspectorComponents = {
  object: ObjectInspector,
  camera: CameraInspector,
  image: ImageInspector,
  light: LightInspector,
  character: CharacterInspector,
  volume: VolumeInspector
}
```

作用：

```text
当前选中对象 type
→ GeneralInspector 查表
→ 渲染对应 Inspector
```

## 5. 底部工具栏：CameraPanelInspector

截图底部红框主要由 `CameraPanelInspector` 渲染。

实现文件：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

它直接读取当前 active camera：

```js
state => ({
  activeCamera: getSceneObjects(state)[getActiveCamera(state)],
  cameraShots: state.cameraShots,
  mainViewCamera: state.mainViewCamera,
})
```

如果没有 active camera，则返回空工具栏：

```js
if (!activeCamera) return <div className="camera-inspector"/>
```

底部工具栏包含：

```text
Roll 控制
Pan / Tilt 控制
Move 控制
Elevate 控制
Lens 控制
Shot Size / Camera Angle / Shot Explorer
```

### 5.1 Roll：镜头滚转

截图底部左侧：

```text
转动: 0°
```

作用：

- 控制镜头沿视线方向左右滚转。
- 类似相机歪头、荷兰角。
- 左右按钮每次改变 `1°`。
- 支持长按连续变化。
- 快捷键：`Z` / `X`。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

UI：

```jsx
<div className="camera-item roll">
  <div className="camera-item-button" {...useLongPress(getValueShifter({ roll: -THREE.Math.DEG2RAD }))}>
    <div className="arrow left"/>
  </div>
  <div className="camera-item-button" {...useLongPress(getValueShifter({ roll: THREE.Math.DEG2RAD }))}>
    <div className="arrow right"/>
  </div>
  <div className="camera-item-label">Roll: { cameraRoll }°</div>
</div>
```

状态更新：

```js
const getValueShifter = (draft) => () => {
  for (let [k, v] of Object.entries(draft)) {
    cameraState[k] += v
  }
  updateObject(activeCamera.id, cameraState)
}
```

快捷键：

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: "cameraRoll",
  keyCustomCheck: (event) => (event.key === 'z' || event.key === 'x') &&
    !event.shiftKey &&
    !event.metaKey &&
    !event.ctrlKey &&
    !event.altKey &&
    !event.mousePressed,
  value: (event) => { rollCamera(event) }
})
```

### 5.2 Pan / Tilt：镜头方向控制

截图底部中间圆形控件：

```text
平移: 90° // 倾斜: -6°
```

这里中文“平移”实际对应 `pan`，即水平转向；`tilt` 是上下俯仰。

作用：

- 拖动圆形控件可以调整相机朝向。
- 横向拖动改变 `rotation`。
- 纵向拖动改变 `tilt`。
- 不是移动相机位置，而是原地转动相机。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

UI：

```jsx
<div className="pan-control" {...getCameraPanEvents()}>
  <div className="pan-control-target"/>
</div>
```

拖动事件来自 `react-use-gesture`：

```js
const getCameraPanEvents = useDrag(({first, last, vxvy }) => {
  dragInfo.current.current = vxvy

  if (first) {
    isDragging.current = true
  } else if (last) {
    isDragging.current = false
  }
})
```

每帧把拖动速度转成 pan / tilt：

```js
const onFrame = () => {
  if (dragInfo.current.prev[0] !== dragInfo.current.current[0] ||
      dragInfo.current.prev[1] !== dragInfo.current.current[1]) {
    const [dx, dy] = dragInfo.current.current

    let newPan = cameraInfo.current.rotation - dx
    let newTilt = cameraInfo.current.tilt - dy

    cameraInfo.current.rotation = newPan
    cameraInfo.current.tilt = newTilt

    let rotation = THREE.Math.degToRad(newPan)
    let tilt = THREE.Math.degToRad(newTilt)

    updateObject(activeCamera.id, {rotation, tilt})
  }

  requestID = requestAnimationFrame(onFrame)
}
```

数据流：

```text
拖动 pan-control
→ useDrag 得到 vxvy
→ requestAnimationFrame 中累积 pan / tilt
→ updateObject(activeCamera.id, { rotation, tilt })
→ 主 Three.js camera 更新
```

### 5.3 Move：相机地面移动

截图底部四向箭头：

```text
移动
```

作用：

- 沿当前相机朝向，在地面平面上前后左右移动相机。
- 上：向前
- 下：向后
- 左：向左
- 右：向右
- 支持长按连续移动。
- 快捷键：`W / A / S / D`。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

UI：

```jsx
<div className="camera-item-button" {...useLongPress(moveCamera([0, -0.1]))}>
  <div className="arrow up"/>
</div>
<div className="camera-item-button" {...useLongPress(moveCamera([-0.1, 0]))}>
  <div className="arrow left"/>
</div>
<div className="camera-item-button" {...useLongPress(moveCamera([0, 0.1]))}>
  <div className="arrow down"/>
</div>
<div className="camera-item-button" {...useLongPress(moveCamera([0.1, 0]))}>
  <div className="arrow right"/>
</div>
```

移动逻辑：

```js
const moveCamera = ([speedX, speedY]) => () => {
  cameraState = CameraControls.getMovedState(cameraState, { x: speedX, y: speedY })
  updateObject(activeCamera.id, cameraState)
}
```

真正根据当前相机 rotation 计算新位置：

- `src/js/shot-generator/CameraControls.js`

```js
CameraControls.getMovedState = (object, movementSpeed) => {
  let loc = new THREE.Vector2(object.x, object.y)
  let result = new THREE.Vector2(
    movementSpeed.x + loc.x,
    movementSpeed.y + loc.y
  ).rotateAround(loc, -object.rotation)

  return {
    ...object,
    x: result.x,
    y: result.y
  }
}
```

关键点：

- Move 只改变 `x / y`。
- 不改变 `z` 高度。
- 移动方向会根据当前相机 `rotation` 旋转。

### 5.4 Elevate：相机升降

截图底部：

```text
提升: 1.78m
```

作用：

- 调整相机高度。
- 上按钮：`z += 0.1`
- 下按钮：`z -= 0.1`
- 支持长按连续变化。
- 快捷键：`R / F`。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

UI：

```jsx
<div className="camera-item elevate">
  <div className="camera-item-button" {...useLongPress(getValueShifter({ z: 0.1 }))}>
    <div className="arrow up"/>
  </div>
  <div className="camera-item-button" {...useLongPress(getValueShifter({ z: -0.1 }))}>
    <div className="arrow down"/>
  </div>
  <div className="camera-item-label">Elevate: { activeCamera.z.toFixed(2) }m</div>
</div>
```

状态更新仍使用：

```js
getValueShifter({ z: 0.1 })
getValueShifter({ z: -0.1 })
```

### 5.5 Lens：镜头焦距 / FOV

截图底部：

```text
镜头: 44.35mm
```

作用：

- 调整相机 FOV。
- UI 显示为焦距 mm。
- 内部状态保存的是 `fov`。
- 左按钮：`fov += 0.2`
- 右按钮：`fov -= 0.2`
- 快捷键：`[` / `]` 按预设焦段切换。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

UI：

```jsx
<div className="camera-item lens">
  <div className="camera-item-button" {...useLongPress(getValueShifter({ fov: 0.2 }))}>
    <div className="arrow left"/>
  </div>
  <div className="camera-item-button" {...useLongPress(getValueShifter({ fov: -0.2 }))}>
    <div className="arrow right"/>
  </div>
  <div className="camera-item-label">Lens: { focalLength.toFixed(2) }mm</div>
</div>
```

焦距显示通过临时 PerspectiveCamera 计算：

```js
const focalLength = useMemo(() => {
  if(!fakeCamera.current) return
  fakeCamera.current.fov = activeCamera.fov
  return fakeCamera.current.getFocalLength()
}, [activeCamera.fov])
```

预设焦段：

```js
const mms = [12, 16, 18, 22, 24, 35, 50, 85, 100, 120, 200, 300, 500]
```

切换逻辑：

```js
const switchCameraFocalLength = useCallback((iterator) => {
  let index = indexIn(fovs, activeCamera.fov)
  let switchTo = index + iterator
  let fov = fovs[Math.max(Math.min(switchTo, fovs.length - 1), 0)]
  fakeCamera.current.fov = fov
  updateObject(activeCamera.id, { fov })
}, [activeCamera])
```

快捷键：

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: "[",
  value: () => switchCameraFocalLength(1)
})

KeyCommandsSingleton.getInstance().addKeyCommand({
  key: "]",
  value: () => switchCameraFocalLength(-1)
})
```

注意：

- 这里 UI 显示焦距 mm。
- 实际状态仍然是 `camera.fov`。
- 右侧主场景最终通过 `camera.fov` 更新 Three.js PerspectiveCamera。

### 5.6 Shot Size：镜头景别

截图底部中间右侧：

```text
Close Up
```

作用：

- 选择自动构图景别。
- 例如 Extreme Close Up、Close Up、Medium Shot、Long Shot 等。
- 选择后会写入 `cameraShots[activeCamera.id].size`。
- 主场景监听 `cameraShots` 后，会自动调整相机位置，让人物符合该景别。

实现位置：

- UI：`src/js/shot-generator/components/CameraPanelInspector/index.js`
- 自动构图逻辑：`src/js/shot-generator/utils/cameraUtils.js`
- 触发调用：`src/js/shot-generator/SceneManagerR3fLarge.js`

选项：

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

写入状态：

```js
const onSetShot = ({size, angle}) => {
  setCameraShot(activeCamera.id, {size, angle})
}
```

Reducer：

- `src/js/shared/reducers/shot-generator.js`

```js
case 'SET_CAMERA_SHOT':
  const camera = getCameraShot(draft, action.payload.cameraId)

  camera.size = action.payload.size || camera.size
  camera.angle = action.payload.angle || camera.angle
  camera.character = action.payload.character
  return
```

自动构图执行：

- `src/js/shot-generator/SceneManagerR3fLarge.js`

```js
if((!cameraShots[key].size && !cameraShots[key].angle) ||
   camera.userData.id !== cameraShots[key].cameraId) continue

setShot({
  camera,
  characters,
  selected,
  updateObject,
  shotSize: cameraShots[key].size,
  shotAngle: cameraShots[key].angle
})
```

构图算法：

- `src/js/shot-generator/utils/cameraUtils.js`

```js
const setShot = ({
  camera,
  characters,
  selected,
  scene,
  updateObject,
  shotAngle,
  shotSize
}) => {
  let {clampedInfo, direction, box} = getShotInfo({
    selected: selected || getClosestCharacter(characters, camera),
    characters,
    shotSize,
    camera
  })

  ...

  camera.position.copy(clampedInfo.position)
  camera.lookAt(clampedInfo.target)
  camera.updateMatrixWorld(true)

  let rot = new THREE.Euler().setFromQuaternion(camera.quaternion, "YXZ")
  updateObject && updateObject(camera.userData.id, {
    x: camera.position.x,
    y: camera.position.z,
    z: camera.position.y,
    rotation: rot.y,
    roll: rot.z,
    tilt: rot.x
  })
}
```

### 5.7 Camera Angle：镜头角度

截图中 Camera Angle 下拉框。

作用：

- 自动设置相机俯仰角度。
- 例如 Bird's Eye、High、Eye、Low、Worm's Eye。
- 与 Shot Size 一起决定自动构图后的相机位置和角度。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`
- `src/js/shot-generator/utils/cameraUtils.js`

选项：

```js
const shotAngles = [
  { value: ShotAngles.BIRDS_EYE, label: "Bird's Eye" },
  { value: ShotAngles.HIGH,      label: "High" },
  { value: ShotAngles.EYE,       label: "Eye" },
  { value: ShotAngles.LOW,       label: "Low" },
  { value: ShotAngles.WORMS_EYE, label: "Worm's Eye" }
]
```

角度映射：

```js
const ShotAnglesInfo = {
  [ShotAngles.BIRDS_EYE]: -30 * THREE.Math.DEG2RAD,
  [ShotAngles.HIGH]: -15 * THREE.Math.DEG2RAD,
  [ShotAngles.EYE]: 0,
  [ShotAngles.LOW]: 30 * THREE.Math.DEG2RAD,
  [ShotAngles.WORMS_EYE]: 45 * THREE.Math.DEG2RAD
}
```

### 5.8 Open Shot Explorer

截图底部：

```text
打开 Shot Explorer
```

作用：

- 打开 Shot Explorer 辅助窗口。
- 通过 Electron IPC 通知主进程。

实现位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js`

```jsx
<div
  className="select-shot-explorer"
  onPointerDown={() => ipcRenderer.send('shot-generator:show:shot-explorer')}
>
  <a className="select-shot-explorer-text">
    {t("shot-generator.camera-panel.open-shot-explorer")}
  </a>
</div>
```

## 6. 底部 BoardInspector：当前 Shot 信息

截图底部中间显示：

```text
Shot 1A
```

实现文件：

- `src/js/shot-generator/components/BoardInspector/index.js`

作用：

- 显示当前 storyboard board 的 shot 编号。
- 如果有 dialogue / action / notes，也会显示。

核心代码：

```jsx
return <div className="column.board-inspector">
  {present(board.shot)
    ? <div className="board-inspector__shot">{ 'Shot ' + board.shot }</div>
    : <div className="board-inspector__shot">Loading …</div>}

  { present(board.dialogue) && <p className="board-inspector__dialogue">...</p> }
  { present(board.action) && <p className="board-inspector__action">...</p> }
  { present(board.notes) && <p className="board-inspector__notes">...</p> }
</div>
```

数据来源：

```js
state => ({
  board: state.board
})
```

## 7. 底部 CamerasInspector：相机切换

截图右下角：

```text
相机  1  2
```

实现文件：

- `src/js/shot-generator/components/CamerasInspector/index.js`

作用：

- 显示所有 camera 的编号。
- 点击编号切换 active camera。
- 当前 active camera 高亮。
- 数字键 `1` 到 `9` 可快速切换相机。

### 7.1 数据来源

只筛选 camera 类型：

```js
const cameraSceneObjectSelector = (state) => {
  const sceneObjects = getSceneObjects(state)
  return Object.values(sceneObjects)
    .filter(object => object.type === "camera")
    .map((object) => ({
      id: object.id,
    }))
}
```

### 7.2 点击切换相机

```js
const onClick = (camera, event) => {
  event.preventDefault()

  undoGroupStart()
  selectObject(camera.id)
  setActiveCamera(camera.id)
  undoGroupEnd()
}
```

### 7.3 快捷键切换相机

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: "cameraSelector",
  keyCustomCheck: (event) => numberCheck(event),
  value: (event) => {
    onCameraSelectByIndex(parseInt(event.key, 10) - 1)
  }
})
```

数字判断：

```js
const numberCheck = (event) => {
  return event.key === '1' ||
    event.key === '2' ||
    event.key === '3' ||
    event.key === '4' ||
    event.key === '5' ||
    event.key === '6' ||
    event.key === '7' ||
    event.key === '8' ||
    event.key === '9'
}
```

数据流：

```text
点击相机编号 / 按数字键
→ selectObject(camera.id)
→ setActiveCamera(camera.id)
→ CameraUpdate 同步主视图 camera
→ 左侧列表 active 点变化
→ 底部相机按钮高亮变化
```

## 8. 底部 GuidesInspector：导览线开关

截图右下角：

```text
导游 / 导览
中心线按钮
三分线按钮
视线按钮
```

实现文件：

- `src/js/shot-generator/components/GuidesInspector/index.js`
- 实际绘制：`src/js/shot-generator/components/GuidesView/index.js`

作用：

- 开关主画面上的构图辅助线。
- 支持三种 guide：
  - center：中心线
  - thirds：三分法网格
  - eyeline：视线参考线

### 8.1 状态来源

```js
state => ({
  center: state.workspace.guides.center,
  thirds: state.workspace.guides.thirds,
  eyeline: state.workspace.guides.eyeline
})
```

### 8.2 点击切换

```jsx
<a
  className={ classNames({ active: center }) }
  onClick={ preventDefault(() => toggleWorkspaceGuide("center")) }
>
  <Icon src="icon-guides-center"/>
</a>

<a
  className={ classNames({ active: thirds }) }
  onClick={ preventDefault(() => toggleWorkspaceGuide("thirds")) }
>
  <Icon src="icon-guides-thirds"/>
</a>

<a
  className={ classNames({ active: eyeline }) }
  onClick={ preventDefault(() => toggleWorkspaceGuide("eyeline")) }
>
  <Icon src="icon-guides-eyeline"/>
</a>
```

状态 action：

- `toggleWorkspaceGuide`
- 来自 `src/js/shared/reducers/shot-generator.js`

## 9. 底部工具栏快捷键汇总

| 功能 | UI 控件 | 快捷键 | 实现文件 |
|---|---|---|---|
| Roll 镜头滚转 | 左右箭头 | `Z` / `X` | `CameraPanelInspector/index.js` |
| Pan / Tilt | 圆形拖拽控件 | 无直接快捷键 | `CameraPanelInspector/index.js` |
| Move 地面移动 | 四向箭头 | `W` / `A` / `S` / `D` | `CameraPanelInspector/index.js`, `CameraControls.js` |
| Elevate 升降 | 上下箭头 | `R` / `F` | `CameraPanelInspector/index.js`, `CameraControls.js` |
| Lens 焦距/FOV | 左右箭头 | `[` / `]` | `CameraPanelInspector/index.js` |
| 切换主视图模式 | 无明显按钮/由预览框扩展按钮触发 | `T` | `CameraPanelInspector/index.js`, `Editor/index.js` |
| 重新选择 active camera | 无 | `Escape` | `CameraPanelInspector/index.js` |
| 相机切换 | 相机编号按钮 | `1` - `9` | `CamerasInspector/index.js` |

## 10. 和主 3D 场景的同步关系

左侧功能区和底部工具栏最终都会更新同一份 camera state。

核心 action：

```js
updateObject(activeCamera.id, patch)
```

同步链路：

```text
左侧 Camera 属性面板 / 底部工具栏
→ updateObject(cameraId, patch)
→ Redux sceneObjects 更新
→ CameraUpdate 读取 activeCamera
→ 更新 Three.js PerspectiveCamera
→ 右侧主场景画面变化
→ 左上角 CameraIcon / 视锥同步变化
```

主相机同步代码：

- `src/js/shot-generator/CameraUpdate.js`

```js
camera.position.x = cameraObject.x
camera.position.y = cameraObject.z
camera.position.z = cameraObject.y
camera.rotation.x = 0
camera.rotation.z = 0
camera.rotation.y = cameraObject.rotation
camera.rotateX(cameraObject.tilt)
camera.rotateZ(cameraObject.roll)
```

注意坐标映射：

```text
业务状态 x → Three.js position.x
业务状态 y → Three.js position.z
业务状态 z → Three.js position.y
```

## 11. 移植到 FlowCanvas 时的建议

### 必须保留的能力

- 左侧对象列表：
  - 选择对象
  - 切换 active camera
  - 锁定/解锁
  - 显示/隐藏
  - 删除

- 左侧 Camera 属性：
  - X / Y / Z
  - rotation
  - roll
  - tilt
  - fov

- 底部 CameraPanel：
  - roll
  - pan / tilt
  - move
  - elevate
  - lens / fov
  - shot size
  - camera angle

- 底部辅助功能：
  - 当前 shot 信息
  - 多相机切换
  - 构图辅助线开关

### 推荐 FlowCanvas 状态字段

```ts
interface DirectorCamera {
  id: string
  type: 'camera'
  visible: boolean
  locked?: boolean

  x: number
  y: number
  z: number

  rotation: number
  roll: number
  tilt: number
  fov: number
}
```

### 推荐事件流

```text
UI 控件交互
→ updateObject(cameraId, patch)
→ store 更新
→ CameraSync 应用到 Three.js camera
→ 3D 画面变化
```

不要让 UI 控件直接操作 Three.js camera。只有右侧主画布拖动时，为了实时流畅，可以先直接操作 Three.js camera，交互结束后再写回 store。

## 12. 关键参考文件索引

| 功能 | 文件 |
|---|---|
| 页面布局 | `src/js/shot-generator/components/Editor/index.js` |
| 左侧总面板 | `src/js/shot-generator/components/ElementsPanel/index.js` |
| Inspector 分发 | `src/js/shot-generator/components/ElementsPanel/Inspector.js` |
| 对象列表 | `src/js/shot-generator/components/ItemList/index.js` |
| 单个对象行 | `src/js/shot-generator/components/ItemList/Item.js` |
| 对象属性总入口 | `src/js/shot-generator/components/InspectedElement/index.js` |
| GeneralInspector 类型分发 | `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js` |
| Camera 属性面板 | `src/js/shot-generator/components/InspectedElement/GeneralInspector/Camera.js` |
| 底部相机工具栏 | `src/js/shot-generator/components/CameraPanelInspector/index.js` |
| 底部 Shot 信息 | `src/js/shot-generator/components/BoardInspector/index.js` |
| 底部相机切换 | `src/js/shot-generator/components/CamerasInspector/index.js` |
| 底部导览线开关 | `src/js/shot-generator/components/GuidesInspector/index.js` |
| 导览线绘制 | `src/js/shot-generator/components/GuidesView/index.js` |
| 相机移动工具函数 | `src/js/shot-generator/CameraControls.js` |
| 自动构图工具 | `src/js/shot-generator/utils/cameraUtils.js` |
| Redux 状态与 action | `src/js/shared/reducers/shot-generator.js` |
