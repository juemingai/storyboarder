# Shot Generator 快捷键实现分析

本文分析 Shot Generator 窗口中快捷键的实现方式、代码入口、事件流和已注册的快捷键。Shot Generator 的快捷键并不全部走同一个系统，而是分成 Electron 菜单快捷键、Renderer 内部快捷键、相机控制键、TransformControl 控制键几类。

## 总体结论

Shot Generator 快捷键主要有两条链路：

1. **Electron 菜单 accelerator 链路**：用于 `Cmd/Ctrl+Z`、`Cmd/Ctrl+D`、`Cmd/Ctrl+G`、`Cmd/Ctrl+B`、`Cmd/Ctrl+K` 等菜单命令。主进程菜单捕获快捷键后，通过 IPC 发到 Shot Generator renderer。
2. **Renderer 内部 KeyCommandsSingleton 链路**：用于 `Delete/Backspace` 删除对象、数字键切换相机、`t` 切换 live/ortho、`z/x` 相机 roll、`[`/`]` 焦距切换、相机 WASD 移动、配件 `Ctrl+E` 切换旋转等。

除此之外，还有一些直接挂在 Three.js 控制器上的快捷键：

- `Ctrl+R` / `Ctrl+T`：TransformControls 切换旋转/平移模式。
- `Ctrl+E`：IK/控制点旋转锁定，配件也复用类似逻辑。
- 数值输入框内的 `Enter` / `Escape`：提交/取消输入。

## 关键代码位置

- Renderer 快捷键注册中心：`src/js/shot-generator/components/KeyHandler/KeyCommandsSingleton.js`
- Renderer 快捷键分发组件：`src/js/shot-generator/components/KeyHandler/index.js`
- KeyHandler 挂载入口：`src/js/shot-generator/components/Editor/index.js`
- Electron 菜单快捷键：`src/js/main/menu.js`
- Shot Generator 主进程 IPC 转发：`src/js/windows/shot-generator/main.js`
- Shot Generator renderer IPC 入口：`src/js/windows/shot-generator/window.js`
- 默认 keymap：`src/js/shared/helpers/defaultKeyMap.js`
- 相机键盘/鼠标控制：`src/js/shot-generator/CameraControls.js`
- 相机面板快捷键：`src/js/shot-generator/components/CameraPanelInspector/index.js`
- 相机编号选择：`src/js/shot-generator/components/CamerasInspector/index.js`
- 选中对象 Drop 命令：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 配件旋转切换：`src/js/shot-generator/components/Three/Attachable.js`
- Three TransformControls：`src/js/shared/IK/utils/TransformControls.js`
- IK TargetControl：`src/js/shared/IK/objects/TargetControl.js`
- 数值输入框键盘处理：`src/js/shot-generator/components/NumberSlider/index.js`

## 架构一：Renderer 内部 KeyCommandsSingleton

### 1. 单例保存两类命令

`KeyCommandsSingleton` 内部维护两个数组：

- `_keyCommands`：renderer 中直接监听 `window.keydown` 的命令。
- `_ipcKeyCommands`：renderer 中通过 `ipcRenderer.on(...)` 监听主进程转发事件的命令。

参考：`src/js/shot-generator/components/KeyHandler/KeyCommandsSingleton.js:2`。

```js
let _keyCommands = []
let _ipcKeyCommands = []
```

注册普通键盘命令：

参考：`src/js/shot-generator/components/KeyHandler/KeyCommandsSingleton.js:66`。

```js
addKeyCommand({key, keyCustomCheck, value}) {
  this.removeKeyCommand(key)
  _keyCommands.push({key, keyCustomCheck, execute: value})
  _updateComponent && _updateComponent({})
}
```

注册 IPC 命令：

参考：`src/js/shot-generator/components/KeyHandler/KeyCommandsSingleton.js:72`。

```js
addIPCKeyCommand({key, keyCustomCheck, value}) {
  let object = _ipcKeyCommands.find(object => object.key === key)
  if(object) {
    let indexOf = _ipcKeyCommands.indexOf(object)
    _ipcKeyCommands.splice(indexOf, 1)
  }
  _ipcKeyCommands.push({key, keyCustomCheck, execute: value})
  _updateComponent && _updateComponent({})
}
```

注意：`addKeyCommand` 里调用的是 `this.removeKeyCommand(key)`，但 `removeKeyCommand` 的签名是 `removeKeyCommand({ key })`。这会导致普通 key command 的去重并不可靠；目前多数命令靠组件 cleanup 和不同 key 名规避问题。

### 2. KeyHandler 统一分发 keydown

`KeyHandler` 在 `Editor` 末尾挂载：

参考：`src/js/shot-generator/components/Editor/index.js:269`。

```jsx
<KeyHandler/>
```

它在 `window` 上注册 `keydown`：

参考：`src/js/shot-generator/components/KeyHandler/index.js:191`。

```js
const onKeyDown = event => {
  if(!KeyCommandsSingleton.getInstance().isEnabledKeysEvents) return
  let keyCommands = KeyCommandsSingleton.getInstance().keyCommands
  for(let i = 0; i < keyCommands.length; i ++ ) {
    let keyCommand = keyCommands[i]
    if(event.key === keyCommand.key) {
      keyCommand.execute(event)
    } else if(keyCommand.keyCustomCheck && keyCommand.keyCustomCheck(event)) {
      keyCommand.execute(event)
    }
  }
}

window.addEventListener('keydown', onKeyDown)
```

匹配规则有两种：

- `event.key === keyCommand.key`
- 或者自定义 `keyCustomCheck(event)` 返回 true

### 3. KeyHandler 统一绑定 IPC 命令

KeyHandler 也把 `_ipcKeyCommands` 绑定到 `ipcRenderer.on(...)`：

参考：`src/js/shot-generator/components/KeyHandler/index.js:171`。

```js
const bindIpcCommands = () => {
  let ipcCommands = keyCommandsInstance.current.ipcKeyCommands
  for(let i = 0; i < ipcCommands.length; i++) {
    ipcRenderer.on(ipcCommands[i].key, ipcCommands[i].execute)
  }
}
```

这就是菜单快捷键最终能执行 renderer 逻辑的关键。

## 架构二：Electron 菜单 accelerator + IPC

Shot Generator 专用菜单在 `src/js/main/menu.js` 的 `shotGeneratorMenu` 中定义。

### 1. 菜单捕获快捷键

例如 Duplicate / Group / Drop / Cycle Shading：

参考：`src/js/main/menu.js:934`。

```js
{
  label: i18n.t('menu.edit.duplicate'),
  accelerator: 'CommandOrControl+d',
  click () {
    send('shot-generator:object:duplicate')
  }
}
```

同一菜单中还有：

- `CommandOrControl+g`：Group / Ungroup
- `CommandOrControl+j`：Open Shot Explorer
- `CommandOrControl+b`：Drop
- `CommandOrControl+k`：Cycle Shading Mode
- `CommandOrControl+=` / `CommandOrControl+-` / `CommandOrControl+0`：UI scale

参考：`src/js/main/menu.js:900` 到 `src/js/main/menu.js:1020`。

### 2. Shot Generator 主进程转发

`src/js/windows/shot-generator/main.js` 接收菜单发出的 IPC，再转发给 Shot Generator renderer：

参考：`src/js/windows/shot-generator/main.js:117`。

```js
ipcMain.on('shot-generator:object:duplicate', () => {
  win.webContents.send('shot-generator:object:duplicate')
})

ipcMain.on('shot-generator:object:group', () => {
  win.webContents.send('shot-generator:object:group')
})

ipcMain.on('shot-generator:view:cycleShadingMode', () => {
  win.webContents.send('shot-generator:view:cycleShadingMode')
})

ipcMain.on('shot-generator:object:drops', () => {
  win.webContents.send('shot-generator:object:drop')
})
```

Undo / Redo 也在这里转发：

```js
ipcMain.on('shot-generator:edit:undo', () => {
  win.webContents.send('shot-generator:edit:undo')
})
ipcMain.on('shot-generator:edit:redo', () => {
  win.webContents.send('shot-generator:edit:redo')
})
```

### 3. Renderer 执行命令

Undo / Redo 直接在 `window.js` 中监听并 dispatch redux-undo：

参考：`src/js/windows/shot-generator/window.js:254`。

```js
ipcRenderer.on('shot-generator:edit:undo', () => {
  store.dispatch(ActionCreators.undo())
})

ipcRenderer.on('shot-generator:edit:redo', () => {
  store.dispatch(ActionCreators.redo())
})
```

Duplicate / Group / Drop / Cycle Shading 则由 KeyHandler 的 IPC 命令执行。

## Electron 菜单快捷键清单

| 快捷键 | 命令 | 入口 | 最终实现 |
|---|---|---|---|
| `Cmd/Ctrl+O` | Open | `src/js/main/menu.js:887` | `send('openDialogue')` |
| `Cmd/Ctrl+Z` | Undo | `src/js/main/menu.js:906` | `window.js` dispatch `ActionCreators.undo()` |
| `Shift+Cmd/Ctrl+Z` | Redo | `src/js/main/menu.js:913` | `window.js` dispatch `ActionCreators.redo()` |
| `Cmd/Ctrl+D` | Duplicate | `src/js/main/menu.js:934` | KeyHandler `duplicateObjects(...)` |
| `Cmd/Ctrl+G` | Group / Ungroup / Merge Groups | `src/js/main/menu.js:942` | KeyHandler `getGroupAction(...)` 后分发 group/ungroup/merge |
| `Cmd/Ctrl+J` | Open Shot Explorer | `src/js/main/menu.js:950` | `shot-generator:show:shot-explorer` |
| `Cmd/Ctrl+B` | Drop selected object/character to ground | `src/js/main/menu.js:968` | `SceneManagerR3fLarge` 的 `onCommandDrop` |
| `Cmd/Ctrl+=` / `Cmd/Ctrl+Plus` | Scale UI up | `src/js/main/menu.js:990` | `Editor` 监听 `shot-generator:menu:view:scale-ui-by` |
| `Cmd/Ctrl+-` | Scale UI down | `src/js/main/menu.js:998` | `Editor` 监听 `shot-generator:menu:view:scale-ui-by` |
| `Cmd/Ctrl+0` | Reset UI scale | `src/js/main/menu.js:1006` | `Editor` 监听 `shot-generator:menu:view:scale-ui-reset` |
| `Cmd/Ctrl+K` | Cycle Shading Mode | `src/js/main/menu.js:1014` | KeyHandler IPC 调用 `cycleShadingMode` |
| `Alt+Cmd+I` | Toggle DevTools | `src/js/main/menu.js:24` | common View submenu |
| `Cmd/Ctrl+M` | Minimize | `src/js/main/menu.js:60` | Electron role |
| `Cmd/Ctrl+W` | Close Window | `src/js/main/menu.js:65` | Electron role |
| `Cmd/Ctrl+R` | Reload dev only | `src/js/main/menu.js:13` | 仅 dev 环境 |

其中 `Cmd/Ctrl+O`、Undo、Redo 的按键来自 `defaultKeyMap`，其他 Shot Generator 专用项中有一些是硬编码 accelerator。

## KeyHandler IPC 命令实现

### 1. Duplicate

注册位置：`src/js/shot-generator/components/KeyHandler/index.js:143`。

```js
keyCommandsInstance.current.addIPCKeyCommand({
  key: 'shot-generator:object:duplicate',
  value: onCommandDuplicate
})
```

执行逻辑：

参考：`src/js/shot-generator/components/KeyHandler/index.js:113`。

```js
const onCommandDuplicate = useCallback(() => {
  if (selections) {
    let selected = (_selectedSceneObject.type === 'group') ? [_selectedSceneObject.id] : selections
    duplicateObjects(
      selected,
      selected.map(THREE.Math.generateUUID)
    )
  }
}, [selections, _selectedSceneObject])
```

### 2. Group / Ungroup / Merge

注册位置：`src/js/shot-generator/components/KeyHandler/index.js:148`。

执行逻辑：根据 `getGroupAction(sceneObjects, selections)` 判断当前应该 group、ungroup 还是 merge。

参考：`src/js/shot-generator/components/KeyHandler/index.js:126`。

```js
const groupAction = getGroupAction(sceneObjects, selections)
if (groupAction.shouldGroup) {
  let group = groupObjects(groupAction.objectsIds)
  selectObject([group.payload.groupId, ...group.payload.ids])
} else if (groupAction.shouldUngroup) {
  selectObject(groupAction.objectsIds)
  ungroupObjects(groupAction.groupsIds[0], groupAction.objectsIds)
} else {
  selectObject(null)
  let group = mergeGroups(groupAction.groupsIds, groupAction.objectsIds)
  selectObject([group.payload.groupIds[0], ...group.payload.ids])
}
```

### 3. Cycle Shading Mode

注册位置：`src/js/shot-generator/components/KeyHandler/index.js:155`。

```js
keyCommandsInstance.current.addIPCKeyCommand({
  key: 'shot-generator:view:cycleShadingMode',
  value: cycleShadingMode
})
```

### 4. Drop

菜单发出 `shot-generator:object:drops`，主进程转为 `shot-generator:object:drop`，`SceneManagerR3fLarge` 注册这个 IPC 命令。

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:289`。

```js
KeyCommandsSingleton.getInstance().addIPCKeyCommand({
  key: 'shot-generator:object:drop',
  value: onCommandDrop
})
```

执行时会遍历当前 selections，对 object / character 调用 drop 逻辑，最后 `updateObjects(changes)` 写回位置：

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:274`。

```js
if (selection.userData.type === 'object') {
  dropObject(selection, droppingPlaces)
  changes[selections[i]] = { x: pos.x, y: pos.z, z: pos.y }
} else if (selection.userData.type === 'character') {
  dropCharacter(selection, droppingPlaces)
  changes[selections[i]] = { x: pos.x, y: pos.z, z: pos.y }
}
```

## Renderer 内部快捷键清单

| 快捷键 | 命令 | 注册位置 | 实现说明 |
|---|---|---|---|
| `Delete` / `Backspace` | 删除选中对象 | `KeyHandler/index.js:162` | 弹确认框；删除 character 时会把 attachables 一起删掉 |
| `1` - `9` | 选择第 N 个相机并设为 activeCamera | `CamerasInspector/index.js:90` | `selectObject(id)` + `setActiveCamera(id)` |
| `t` | 切换主视图 live / ortho | `CameraPanelInspector/index.js:91` | `setMainViewCamera(...)` |
| `Escape` | 选择 activeCamera | `CameraPanelInspector/index.js:98` | `selectObject(activeCamera)` |
| `z` | 相机 roll -1 度 | `CameraPanelInspector/index.js:170` | 不能带 Shift/Cmd/Ctrl/Alt，且鼠标拖拽中不触发 |
| `x` | 相机 roll +1 度 | `CameraPanelInspector/index.js:170` | 同上 |
| `[` | 切到更广/更小焦距档位 | `CameraPanelInspector/index.js:184` | `switchCameraFocalLength(1)` |
| `]` | 切到更长/更大焦距档位 | `CameraPanelInspector/index.js:191` | `switchCameraFocalLength(-1)` |
| `W` / `↑` | 相机向前移动 | `CameraControls.js:210` | 持续按键，帧循环更新 |
| `S` / `↓` | 相机向后移动 | `CameraControls.js:214` | 持续按键，帧循环更新 |
| `A` / `←` | 相机向左移动 | `CameraControls.js:212` | 持续按键，帧循环更新 |
| `D` / `→` | 相机向右移动 | `CameraControls.js:216` | 持续按键，帧循环更新 |
| `R` | 相机升高 | `CameraControls.js:218` | 持续按键，帧循环更新 |
| `F` | 相机降低 | `CameraControls.js:219` | 持续按键，帧循环更新，不低于 0 |
| `Shift` | 相机键盘移动加速；鼠标 Dolly | `CameraControls.js:207`、`CameraControls.js:541` | 按住时 `runMode` 加速 |
| `Ctrl` + 鼠标拖拽 / 右键拖拽 | 相机 Orbit | `CameraControls.js:379` | 围绕目标点或视野中心旋转 |
| `Alt` + 鼠标拖拽 / 中键拖拽 | 相机 Truck/Pedestal | `CameraControls.js:477` 附近 | 平移相机 |
| `Shift + Alt` + 鼠标拖拽 / `Shift + 中键` | Dolly Zoom | `CameraControls.js:334` | 同步调整位置和 fov |
| 鼠标滚轮 | 改变 fov/zoomSpeed | `CameraControls.js:260` | 限制 fov 在 1 到 90 |
| `Ctrl+E` | 配件移动/旋转控制切换 | `Attachable.js:206`、`Attachable.js:351` | 仅配件选中时注册 |
| `Ctrl+R` | TransformControls 切换 Rotate | `TransformControls.js:594` | 直接监听 window keydown |
| `Ctrl+T` | TransformControls 切换 Translate | `TransformControls.js:599` | `rotationOnly` 时禁用 |
| `Ctrl+E` | IK TargetControl 锁定/解锁旋转 | `TargetControl.js:99` | 选中控制点时生效 |
| `Enter` | NumberSlider 文本输入提交 | `NumberSlider/index.js:163` | 校验并 `onSetValue` |
| `Escape` | NumberSlider 文本输入取消 | `NumberSlider/index.js:159` | 恢复原值并退出输入 |

## 删除快捷键的实现细节

`Delete` / `Backspace` 不是 Electron 菜单的 `role: delete` 实现，而是 Renderer 内部命令。

注册位置：`src/js/shot-generator/components/KeyHandler/index.js:162`。

```js
keyCommandsInstance.current.addKeyCommand({
  key: 'removeElement',
  keyCustomCheck: (event) => event.key === 'Backspace' || event.key === 'Delete',
  value: deleteSelectedObject
})
```

删除前会校验对象类型：

参考：`src/js/shot-generator/components/KeyHandler/index.js:10`。

```js
const canDelete = (sceneObject, activeCamera) =>
  sceneObject.type === 'object' ||
  sceneObject.type === 'character' ||
  sceneObject.type === 'volume' ||
  sceneObject.type === 'light' ||
  sceneObject.type === 'image' ||
  (sceneObject.type === 'camera' && sceneObject.id !== activeCamera)
```

删除人物时，额外把绑定在人物上的 attachable 一起加入删除列表：

参考：`src/js/shot-generator/components/KeyHandler/index.js:95`。

```js
if(sceneObject.type === 'character') {
  let attachableIds = Object.values(sceneObjects)
    .filter(obj => obj.attachToId === selections[i])
    .map(obj => obj.id)
  objectsToDelete = attachableIds.concat(objectsToDelete)
}
```

## 相机控制快捷键实现

相机控制是一个特殊 case：`CameraControls` 把自己的 `onKeyDown` 注册成 `keyCustomCheck`，并且 `value` 是空函数。

参考：`src/js/shot-generator/CameraControls.js:59`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: 'camera-controls',
  keyCustomCheck: this.onKeyDown,
  value: () => {}
})
window.addEventListener('keyup', this.onKeyUp, false)
```

这里的 `onKeyDown` 本身会修改 `moveForward/moveBackward/...` 等状态，并返回一个 truthy/falsy 值并不重要；KeyHandler 随后执行空函数。真正的移动发生在 `CameraControls.update(delta, state)` 的帧循环里。

按键映射：

参考：`src/js/shot-generator/CameraControls.js:206`。

```js
case 38: // up
case 87: // W
  this.moveForward = true
  break
case 37: // left
case 65: // A
  this.moveLeft = true
  break
case 40: // down
case 83: // S
  this.moveBackward = true
  break
case 39: // right
case 68: // D
  this.moveRight = true
  break
case 82: // R
  this.moveUp = true
  break
case 70: // F
  this.moveDown = true
  break
```

帧循环根据这些布尔值更新相机位置：

参考：`src/js/shot-generator/CameraControls.js:540`。

```js
if (this.moveForward || this.moveBackward || this.moveLeft || this.moveRight || this.moveUp || this.moveDown) {
  if (this.runMode) {
    this.movementSpeed += ...
  } else {
    this.movementSpeed += ...
  }
}
```

## 相机面板快捷键实现

`CameraPanelInspector` 注册了几个 UI/镜头相关快捷键：

参考：`src/js/shot-generator/components/CameraPanelInspector/index.js:91`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: 't',
  value: () => setMainViewCamera(mainViewCamera === 'ortho' ? 'live' : 'ortho')
})
```

参考：`src/js/shot-generator/components/CameraPanelInspector/index.js:98`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: 'Escape',
  value: () => selectObject(activeCamera)
})
```

Roll：

参考：`src/js/shot-generator/components/CameraPanelInspector/index.js:170`。

```js
keyCustomCheck: (event) => (event.key === 'z' || event.key === 'x') &&
  !event.shiftKey && !event.metaKey && !event.ctrlKey && !event.altKey && !event.mousePressed
```

焦距切换：

参考：`src/js/shot-generator/components/CameraPanelInspector/index.js:184`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({ key: '[', value: () => switchCameraFocalLength(1) })
KeyCommandsSingleton.getInstance().addKeyCommand({ key: ']', value: () => switchCameraFocalLength(-1) })
```

## 相机数字键选择

`CamerasInspector` 使用 `1` 到 `9` 选择对应序号的相机，并设为 active camera。

参考：`src/js/shot-generator/components/CamerasInspector/index.js:38`。

```js
const numberCheck = (event) => {
  return event.key === '1' || event.key === '2' || ... || event.key === '9'
}
```

注册：

参考：`src/js/shot-generator/components/CamerasInspector/index.js:90`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: 'cameraSelector',
  keyCustomCheck: (event) => numberCheck(event),
  value: (event) => { onCameraSelectByIndex(parseInt(event.key, 10) - 1) }
})
```

## 配件快捷键

当 attachable 被选中时，`Attachable` 注册 `Ctrl+E`，用于切换配件的移动/旋转操控状态。

注册位置：`src/js/shot-generator/components/Three/Attachable.js:203`。

```js
KeyCommandsSingleton.getInstance().addKeyCommand({
  key: 'Switch attachables to rotation',
  value: switchManipulationState,
  keyCustomCheck: controlPlusRCheck
})
```

检测逻辑：`src/js/shot-generator/components/Three/Attachable.js:351`。

```js
const controlPlusRCheck = (event) => {
  event.stopPropagation()
  if(event.ctrlKey && event.key === 'e'){
    event.stopPropagation()
    return true
  }
}
```

命名上叫 `controlPlusRCheck`，但实际检测的是 `Ctrl+E`。

## TransformControls / IK 控制快捷键

Three.js TransformControls 直接监听 keydown，不走 KeyCommandsSingleton。

参考：`src/js/shared/IK/utils/TransformControls.js:594`。

```js
if(event.ctrlKey) {
  if(event.key === 'r') {
    event.stopPropagation()
    scope.setMode('rotate')
  }
  if(event.key === 't' && !this.rotationOnly) {
    event.stopPropagation()
    scope.setMode('translate')
  }
}
```

IK TargetControl 也直接监听 keydown，用 `Ctrl+E` 切换旋转锁定：

参考：`src/js/shared/IK/objects/TargetControl.js:99`。

```js
if(event.ctrlKey) {
  if(event.key === 'e') {
    event.stopPropagation()
    this.isRotationLocked = !this.isRotationLocked
    this.updateInitialPosition()
  }
}
```

## 输入框对全局快捷键的屏蔽

`NumberSlider` 进入文本输入模式时，会关闭全局 KeyCommandsSingleton 的 keydown 分发，避免按数字、Delete、Enter 等误触发全局命令。

参考：`src/js/shot-generator/components/NumberSlider/index.js:194`。

```js
KeyCommandSingleton.getInstance().isEnabledKeysEvents = !isTextInput
```

文本输入内部自己处理：

参考：`src/js/shot-generator/components/NumberSlider/index.js:159`。

```js
if (event.key === 'Escape') {
  setTextInput(false)
  setTextInputValue(getFormattedInputValue(sliderValue, formatter))
} else if (event.key === 'Enter') {
  // parse/constraint/onSetValue
  setTextInput(false)
}
```

## 默认 keymap 与可配置性

全局默认 keymap 在 `src/js/shared/helpers/defaultKeyMap.js`。Shot Generator 菜单复用了其中一部分：

- `menu:file:open` -> `CommandOrControl+o`
- `menu:edit:undo` -> `CommandOrControl+z`
- `menu:edit:redo` -> `Shift+CommandOrControl+z`
- `menu:view:zoom-out` -> `CommandOrControl+-`
- `menu:view:toggle-developer-tools` -> `Alt+Command+i`
- `menu:window:minimize` -> `CommandOrControl+m`
- `menu:window:close` -> `CommandOrControl+w`

参考：`src/js/shared/helpers/defaultKeyMap.js:21`。

用户可通过偏好设置中的 keymap 文件修改这些 Electron 菜单 accelerator；但 Renderer 内部硬编码的快捷键（比如 `t`、`z/x`、`[`/`]`、`1-9`、`WASD`、`Ctrl+E`）没有接入 `defaultKeyMap`。

## 设计和维护注意点

- Shot Generator 快捷键分散在多个组件中，没有集中注册表；排查冲突时需要搜 `addKeyCommand`、`addIPCKeyCommand`、`keydown`、`accelerator`。
- 菜单快捷键和 renderer 快捷键是两套机制；`Cmd/Ctrl+D` 这类先走 Electron 菜单，再经 IPC 到 renderer。
- `KeyCommandsSingleton.addKeyCommand` 的去重调用参数不匹配，可能导致重复注册风险。
- `CameraControls` 通过 `keyCustomCheck` 执行状态更新，但 `value` 是空函数；这是一个副作用式用法。
- NumberSlider 文本输入会全局禁用 KeyCommandsSingleton，但不会影响直接挂在 window/domElement 上的 TransformControls 类 keydown。
- macOS 上 `event.metaKey` 会让 `CameraControls.onKeyDown` 直接 return，避免 Cmd+R/Cmd+D 这类菜单快捷键污染相机移动状态。
