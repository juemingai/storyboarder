# Shot Generator 左上相机预览与右侧主场景同步分析

本文分析 Shot Generator 中左上角相机预览框（top-down / 俯视预览）和右侧主场景之间的比例关系、坐标同步方式、拖动相机时右侧主场景如何变化，以及左上角拖动是否存在边界限制。

## 结论摘要

左上角预览框和右侧主场景使用同一份 Redux `sceneObjects` 数据，但用不同相机和不同渲染方式展示：

- 左上角预览框是正交俯视图，用于查看和拖动相机、人物、物体等场景对象。
- 右侧主场景是当前 `activeCamera` 的真实透视相机视图。
- 左上角拖动相机图标时，更新的是相机对象在 `sceneObjects` 中的 `x/y` 世界坐标。
- 右侧主场景读取同一个 active camera 对象，所以会按世界坐标 **1:1 同步移动**。
- 拖动比例不是固定“1 像素 = N 米”，而是由左上角正交相机当前自动适配出来的可视范围决定。
- 代码层面没有对左上角相机拖动设置场景边界；相机可以被拖出地面、房间或当前可见区域。

## 两个视图分别由谁渲染

页面主布局在 `Editor` 中组合左上角预览和右侧主场景。

| 区域 | 文件位置 | 说明 |
| --- | --- | --- |
| 左上角预览 Canvas | `src/js/shot-generator/components/Editor/index.js:199` 到 `src/js/shot-generator/components/Editor/index.js:214` | 使用 `orthographic={true}`，渲染 `SceneManagerR3fSmall`。 |
| 右侧主场景 Canvas | `src/js/shot-generator/components/Editor/index.js:243` 到 `src/js/shot-generator/components/Editor/index.js:248` | 渲染 `SceneManagerR3fLarge`，默认使用当前 live camera。 |
| 左上预览组件 | `src/js/shot-generator/SceneManagerR3fSmall.js:29` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:42` | 连接 Redux，读取 `sceneObjects`、`world`、`aspectRatio`、`selections`。 |
| 右侧主场景组件 | `src/js/shot-generator/SceneManagerR3fLarge.js:75` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:97` | 连接 Redux，读取同一份 `sceneObjects`、`world`、`selections`、`cameraShots`。 |

默认 `mainViewCamera === 'live'` 时：

- 左上角显示 `SceneManagerR3fSmall` 自己的正交俯视场景。
- 右侧显示 `SceneManagerR3fLarge` 的 live 透视相机视图。

对应逻辑：

- `src/js/shot-generator/components/Editor/index.js:210`
- `src/js/shot-generator/components/Editor/index.js:211`
- `src/js/shot-generator/components/Editor/index.js:245`

## 坐标体系关系

Shot Generator 业务数据中，场景对象坐标一般使用：

```js
{
  x,
  y,
  z
}
```

但渲染到 Three.js 时，项目采用轴映射：

```js
Three.js position = [x, z, y]
```

也就是：

| Shot Generator 业务坐标 | Three.js 坐标 | 含义 |
| --- | --- | --- |
| `x` | `position.x` | 水平横向 |
| `y` | `position.z` | 地面平面纵深 |
| `z` | `position.y` | 高度 |

默认相机图标也按这个规则渲染：

- `src/js/shot-generator/components/Three/Icons/CameraIcon.js:98` 到 `src/js/shot-generator/components/Three/Icons/CameraIcon.js:103`

关键代码：

```js
const { x, y, z} = sceneObject
return <group position={ [x, z, y] }>
```

右侧主场景的真实 Three.js camera 也按同样规则同步：

- `src/js/shot-generator/CameraUpdate.js:21` 到 `src/js/shot-generator/CameraUpdate.js:23`

关键代码：

```js
camera.position.x = cameraObject.x
camera.position.y = cameraObject.z
camera.position.z = cameraObject.y
```

因此，左上角拖动相机后，本质上是更新业务坐标里的 `x/y`；右侧主场景会把这两个值映射到 Three.js camera 的 `x/z` 平面位置。

## 左上角移动相机，右侧主场景移动多少

### 世界坐标层面：1:1

如果左上角拖动相机后，Redux 中相机对象变化为：

```js
{
  x: oldX + 1,
  y: oldY + 2
}
```

右侧主场景中的 active camera 也会同步为：

```js
camera.position.x = oldX + 1
camera.position.z = oldY + 2
```

所以从世界坐标角度看，二者是 **1:1 同步**。

需要注意：右侧画面看到的视觉变化不是“右侧画布里的相机图标移动”，而是“当前透视相机位置改变后，看到的场景视角变化”。如果场景里只有地面网格，拖动相机时会表现为网格透视位置发生移动。

### 屏幕像素层面：不是固定比例

左上角预览框里的拖动像素不直接等于固定米数。换算关系取决于左上角正交相机当前的可视范围。

核心自动适配函数是：

- `src/js/shot-generator/SceneManagerR3fSmall.js:132` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:200`

其中会根据所有对象的位置计算 `minMax`，再设置正交相机范围：

- `src/js/shot-generator/SceneManagerR3fSmall.js:192` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:200`

关键代码：

```js
camera.position.x = minMax[0]+((minMax[1]-minMax[0])/2)
camera.position.z = minMax[2]+((minMax[3]-minMax[2])/2)
camera.left = -(minMax[1]-minMax[0])/2
camera.right = (minMax[1]-minMax[0])/2
camera.top = (minMax[3]-minMax[2])/2
camera.bottom = -(minMax[3]-minMax[2])/2
```

因此可以用下面公式理解左上角拖动比例：

```txt
水平每像素对应世界单位 = 当前正交相机可视宽度 / 左上角预览框像素宽度
垂直每像素对应世界单位 = 当前正交相机可视高度 / 左上角预览框像素高度
```

### 默认空场景时的大致比例

当场景里只有一个相机时，`autofitOrtho()` 会给范围加额外 padding：

- 只有一个对象时加 padding：`src/js/shot-generator/SceneManagerR3fSmall.js:156` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:163`
- 通用 padding：`src/js/shot-generator/SceneManagerR3fSmall.js:165` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:169`

默认只有一个相机时，正交视图范围大约是 `8m x 8m`。渲染器在非 `renderData` 模式下设置为 `300 x 300`：

- `src/js/shot-generator/SceneManagerR3fSmall.js:287` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:290`

所以默认空场景下可以粗略估算：

```txt
1 像素约等于 8 / 300 = 0.0267 米
100 像素约等于 2.67 米
```

但这只是默认空场景的近似值。添加人物、物体、灯光、图片或移动对象后，`autofitOrtho()` 会重新适配整体范围，比例会变化。

## 左上角拖动相机的数据流

### 1. 左上角监听 pointer 事件

左上角预览框给 canvas DOM 绑定 pointer 事件：

- `src/js/shot-generator/SceneManagerR3fSmall.js:269` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:278`

包括：

```js
pointerdown -> intersectLogic
pointermove -> onPointerMove
pointerup -> onPointerUp
```

### 2. pointer down 命中相机图标并选中

命中逻辑通过 raycaster 找到对象：

- `src/js/shot-generator/SceneManagerR3fSmall.js:225` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:263`

真正开始拖动在：

- `src/js/shot-generator/SceneManagerR3fSmall.js:96` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:109`

关键逻辑：

```js
selectObject(match.userData.id)
if(match.userData.type === "camera") {
  setActiveCamera(match.userData.id)
}
draggedObject.current = match
prepareDrag(...)
```

这说明点击左上角相机图标时：

- 会选中该相机。
- 如果是 camera，会把它设置为 active camera。
- 随后进入拖动准备阶段。

### 3. pointer move 计算新位置

拖动时，`SceneManagerR3fSmall` 调用拖拽管理器：

- `src/js/shot-generator/SceneManagerR3fSmall.js:111` 到 `src/js/shot-generator/SceneManagerR3fSmall.js:116`

关键代码：

```js
drag({ x, y }, draggedObject.current, camera, selections)
updateStore(updateObjects)
```

### 4. 拖拽管理器用 raycaster + 平面求交

拖拽管理器在：

- `src/js/shot-generator/hooks/use-dragging-manager.js:13` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:55`
- `src/js/shot-generator/hooks/use-dragging-manager.js:57` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:102`

左上角传入 `useDraggingManager(true)`：

- `src/js/shot-generator/SceneManagerR3fSmall.js:64`

因此 prepare 阶段会走 `useIcons` 分支：

- `src/js/shot-generator/hooks/use-dragging-manager.js:21` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:23`

关键代码：

```js
plane.current.setFromNormalAndCoplanarPoint(
  camera.position.clone().normalize(),
  target.position
)
```

拖动阶段会把射线与该平面求交，然后把交点转成对象新位置：

- `src/js/shot-generator/hooks/use-dragging-manager.js:78` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:96`

关键代码：

```js
let { x, z } = intersection.current.clone()
  .sub(offsets.current[selection])
  .setY(0)

target.position.set(x, target.position.y, z)
objectChanges.current[selection] = { x, y: z }
```

这里 `x` 和 `z` 是 Three.js 地面平面的横向坐标，写回 Redux 时变为业务坐标：

```js
{
  x,
  y: z
}
```

### 5. 更新 Redux

拖动过程中和结束时都会调用 `updateObjects`：

- 拖动中：`src/js/shot-generator/hooks/use-dragging-manager.js:104` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:109`
- 拖动结束：`src/js/shot-generator/hooks/use-dragging-manager.js:111` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:123`

`UPDATE_OBJECTS` reducer 负责写回场景对象：

- `src/js/shared/reducers/shot-generator.js:1079` 到 `src/js/shared/reducers/shot-generator.js:1089`

关键代码：

```js
draft[key].x = value.x ? value.x : draft[key].x
draft[key].y = value.y ? value.y : draft[key].y
draft[key].z = value.z ? value.z : draft[key].z
draft[key].rotation = value.rotation ? value.rotation : draft[key].rotation
```

## 右侧主场景如何同步

右侧主场景内部挂载了 `CameraUpdate`：

- `src/js/shot-generator/SceneManagerR3fLarge.js:345` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:347`

`CameraUpdate` 从 Redux 中读取当前 active camera：

- `src/js/shot-generator/CameraUpdate.js:15` 到 `src/js/shot-generator/CameraUpdate.js:18`

再把业务坐标和旋转写到 Three.js camera：

- `src/js/shot-generator/CameraUpdate.js:19` 到 `src/js/shot-generator/CameraUpdate.js:32`

关键代码：

```js
camera.position.x = cameraObject.x
camera.position.y = cameraObject.z
camera.position.z = cameraObject.y
camera.rotation.y = cameraObject.rotation
camera.rotateX(cameraObject.tilt)
camera.rotateZ(cameraObject.roll)
```

因此左上角预览拖动相机后，Redux 中相机对象更新，右侧主场景的相机位置随之更新。

## 左上角拖动相机是否有边界

代码层面没有场景边界限制。

左上角拖动时，只检查目标对象是否可拖：

- `src/js/shot-generator/SceneManagerR3fSmall.js:101`

关键逻辑：

```js
if(!match || !match.userData || match.userData.locked || match.userData.blocked) return
```

拖动管理器里也只跳过 locked / blocked 对象：

- `src/js/shot-generator/hooks/use-dragging-manager.js:80` 到 `src/js/shot-generator/hooks/use-dragging-manager.js:82`

关键逻辑：

```js
if (!target || target.userData.locked || target.userData.blocked) continue
```

没有看到类似下面的 clamp 限制：

```js
x = clamp(x, minX, maxX)
y = clamp(y, minY, maxY)
```

所以：

- 不限制相机必须在地面网格内。
- 不限制相机必须在房间范围内。
- 不限制相机必须在当前预览框可见范围内。
- 不限制相机最大或最小 `x/y`。

唯一和高度相关的边界存在于右侧主场景的相机控制逻辑里，不是左上角拖动相机图标的边界。例如右侧主场景通过键盘/控制器下降相机时，会限制 `z >= 0`：

- `src/js/shot-generator/CameraControls.js:601` 到 `src/js/shot-generator/CameraControls.js:604`

但左上角拖动相机图标只改 `x/y`，不改 `z`，因此不受这个高度限制影响。

## 左上角预览框自身会自动缩放 / 适配

虽然拖动没有业务边界，但左上角预览框会自动重新适配对象范围。

`autofitOrtho()` 的依赖包括：

- `sceneObjects`
- `aspectRatio`
- `fontMesh`
- `renderData`

位置：

- `src/js/shot-generator/SceneManagerR3fSmall.js:210`

这意味着拖动相机后，`sceneObjects` 变化，左上角正交相机范围可能重新计算，所以用户看到的预览比例可能会变化。

此外 `CameraIcon` 在检测到主相机位置变化后也会触发 `autofitOrtho()`：

- `src/js/shot-generator/components/Three/Icons/CameraIcon.js:47` 到 `src/js/shot-generator/components/Three/Icons/CameraIcon.js:60`

关键逻辑：

```js
let isPositionChanged = !ref.current.position.equals(mainCamera.position)
if(isPositionChanged) {
  ref.current.position.copy(mainCamera.position)
  props.autofitOrtho()
}
```

这保证当主视图相机位置变化时，左上角相机图标和俯视预览范围也能同步。

## 已发现的边缘问题

`UPDATE_OBJECTS` reducer 使用 truthy 判断写入字段：

- `src/js/shared/reducers/shot-generator.js:1084`
- `src/js/shared/reducers/shot-generator.js:1085`
- `src/js/shared/reducers/shot-generator.js:1086`
- `src/js/shared/reducers/shot-generator.js:1087`

代码：

```js
draft[key].x = value.x ? value.x : draft[key].x
draft[key].y = value.y ? value.y : draft[key].y
draft[key].z = value.z ? value.z : draft[key].z
draft[key].rotation = value.rotation ? value.rotation : draft[key].rotation
```

这会导致一个潜在问题：如果拖动结果刚好是 `0`，因为 `0` 是 falsy，字段可能不会被写入。比如从 `x = 1` 拖回精确 `x = 0` 时，`x` 可能保持旧值。

更稳妥的写法应该是判断字段是否存在：

```js
if (value.hasOwnProperty('x')) draft[key].x = value.x
if (value.hasOwnProperty('y')) draft[key].y = value.y
if (value.hasOwnProperty('z')) draft[key].z = value.z
if (value.hasOwnProperty('rotation')) draft[key].rotation = value.rotation
```

或使用 `value.x != null` 一类判断。

## 快速回答

| 问题 | 回答 |
| --- | --- |
| 左上角和右侧主场景是什么关系？ | 同一份 `sceneObjects` 数据的俯视正交预览和 active camera 透视视图。 |
| 左上角移动相机，右侧移动多少？ | 世界坐标 1:1；左上角更新 `sceneObject.x/y`，右侧 active camera 同步这些值。 |
| 左上角每拖 1 像素是多少米？ | 不固定，取决于 `autofitOrtho()` 当前算出的正交可视范围。默认空场景粗略约 `0.0267m/px`。 |
| 左上角移动相机有没有边界？ | 代码层面没有 `x/y` 边界；只会跳过 locked / blocked 对象。 |
| 会不会限制在地面或房间内？ | 不会。 |
| 拖动会不会改变相机高度？ | 不会。左上角拖动只写回 `x/y`，不改 `z`。 |
