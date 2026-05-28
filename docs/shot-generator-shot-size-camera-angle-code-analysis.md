# Shot Generator：Shot Size 与 Camera Angle 代码实现分析

## 1. 功能入口概览

截图中底部相机控制栏的 `Shot Size` 和 `Camera Angle` 两个下拉框，属于 Shot Generator 的相机自动构图功能。它们并不直接在 UI 层修改相机位置，而是先把用户选择写入 Redux 的 `cameraShots` 状态；随后 3D 场景组件监听 `cameraShots` 变化，调用 `cameraUtils.setShot` 根据人物骨骼包围盒、当前相机视角、镜头景别和镜头角度重新计算相机 transform。

核心链路如下：

```text
CameraPanelInspector 下拉框
  -> setCameraShot(activeCamera.id, { size, angle })
  -> cameraShotsReducer 保存到 state.cameraShots[cameraId]
  -> SceneManagerR3fLarge 监听 cameraShots
  -> setShot({ camera, characters, selected, shotSize, shotAngle })
  -> cameraUtils 计算相机 position/lookAt
  -> updateObject(cameraId, { x, y, z, rotation, roll, tilt })
```

## 2. 相关文件与代码位置

| 文件 | 代码位置 | 作用 |
| --- | --- | --- |
| `src/js/shot-generator/components/CameraPanelInspector/index.js` | `src/js/shot-generator/components/CameraPanelInspector/index.js:76` | 从 `cameraShots[activeCamera.id]` 读取当前相机的 shot 信息。 |
| `src/js/shot-generator/components/CameraPanelInspector/index.js` | `src/js/shot-generator/components/CameraPanelInspector/index.js:77` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:89` | 使用本地 state 缓存当前 `size` 和 `angle`，并在 Redux 状态变化或切换相机时同步显示值。 |
| `src/js/shot-generator/components/CameraPanelInspector/index.js` | `src/js/shot-generator/components/CameraPanelInspector/index.js:244` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:246` | 定义 `onSetShot`，把下拉框选择派发为 `SET_CAMERA_SHOT`。 |
| `src/js/shot-generator/components/CameraPanelInspector/index.js` | `src/js/shot-generator/components/CameraPanelInspector/index.js:248` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:267` | 定义 UI 可选的 Shot Size 和 Camera Angle 列表。 |
| `src/js/shot-generator/components/CameraPanelInspector/index.js` | `src/js/shot-generator/components/CameraPanelInspector/index.js:328` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:341` | 渲染两个 `Select` 下拉框，并在选择时保留另一个维度的值。 |
| `src/js/shot-generator/components/Select/index.js` | `src/js/shot-generator/components/Select/index.js:27` 到 `src/js/shot-generator/components/Select/index.js:39` | 通用下拉组件，基于 `react-select`，使用 `label` 作为 placeholder，禁用搜索。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:403` 到 `src/js/shared/reducers/shot-generator.js:412` | `getCameraShot` 初始化单个相机的 `{ size, angle, cameraId }` 记录。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:751` | 初始状态中 `cameraShots` 为空对象。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:754` 到 `src/js/shared/reducers/shot-generator.js:763` | `cameraShotsReducer` 处理 `SET_CAMERA_SHOT`，保存 `size` / `angle` / `character`。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:1682` 到 `src/js/shared/reducers/shot-generator.js:1685` | 根 reducer 把 `cameraShotsReducer` 接入全局 state。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:1757` | action creator：`setCameraShot(cameraId, values)`。 |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | `src/js/shot-generator/SceneManagerR3fLarge.js:236` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:258` | 监听 `cameraShots`，找到当前相机对应记录后调用 `setShot` 应用自动构图。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:29` 到 `src/js/shot-generator/utils/cameraUtils.js:42` | 定义 Shot Size 枚举值。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:44` 到 `src/js/shot-generator/utils/cameraUtils.js:58` | 定义 Camera Angle 枚举值和角度映射。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:60` 到 `src/js/shot-generator/utils/cameraUtils.js:145` | 定义每种 Shot Size 对应的骨骼范围、画面裁切和特殊构图参数。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:155` 到 `src/js/shot-generator/utils/cameraUtils.js:228` | `getShotInfo` 根据景别生成构图 box、目标点和初始相机方向。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:230` 到 `src/js/shot-generator/utils/cameraUtils.js:269` | `getShotBox` 根据人物骨骼计算需要框入画面的 `THREE.Box3`。 |
| `src/js/shot-generator/utils/cameraUtils.js` | `src/js/shot-generator/utils/cameraUtils.js:303` 到 `src/js/shot-generator/utils/cameraUtils.js:354` | `setShot` 是最终执行函数：应用景别和角度，写回相机位置与旋转。 |
| `src/js/shot-explorer/ShotMaker.js` | `src/js/shot-explorer/ShotMaker.js:130` 到 `src/js/shot-explorer/ShotMaker.js:142` | Shot Explorer 也复用同一组 `ShotSizes`、`ShotAngles` 和 `setShot` 生成随机镜头。 |

## 3. UI 层实现

### 3.1 当前相机 shot 状态读取

文件：`src/js/shot-generator/components/CameraPanelInspector/index.js`

`CameraPanelInspector` 通过 `connect` 读取：

- `activeCamera`：当前激活相机对象。
- `cameraShots`：按相机 id 存储的自动构图选择。
- `mainViewCamera`：主视图模式。

关键位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js:55` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:56`
- `src/js/shot-generator/components/CameraPanelInspector/index.js:76`

```js
const shotInfo = cameraShots[activeCamera.id] || {}
```

这里的 `shotInfo` 是当前相机的自动构图配置。没有配置时为空对象，因此 UI 显示 `Shot Size` / `Camera Angle` placeholder。

### 3.2 下拉框显示值同步

关键位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js:77` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:89`

组件用本地 state 保存当前下拉值：

```js
const [currentShotSize, setCurrentShotSize] = useState(shotInfo.size)
const [currentShotAngle, setCurrentShotAngle] = useState(shotInfo.angle)
```

两个 `useEffect` 分别监听 `shotInfo.size` 和 `shotInfo.angle`。当用户切换相机，或 reducer 写入新的 shot 信息后，本地显示值会重新同步。

### 3.3 Shot Size 选项

关键位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js:248` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:259`

UI 下拉框暴露 10 个景别：

| UI label | 存储值 |
| --- | --- |
| Extreme Close Up | `ShotSizes.EXTREME_CLOSE_UP` / `Extremely close up` |
| Very Close Up | `ShotSizes.VERY_CLOSE_UP` / `Very close up` |
| Close Up | `ShotSizes.CLOSE_UP` / `Close up` |
| Medium Close Up | `ShotSizes.MEDIUM_CLOSE_UP` / `Medium close up` |
| Bust | `ShotSizes.BUST` / `Bust` |
| Medium Shot | `ShotSizes.MEDIUM` / `Medium` |
| Medium Long Shot | `ShotSizes.MEDIUM_LONG` / `Medium long` |
| Long Shot / Wide | `ShotSizes.LONG` / `Long` |
| Extreme Long Shot | `ShotSizes.EXTREME_LONG` / `Extremely long` |
| Establishing Shot | `ShotSizes.ESTABLISHING` / `Establishing` |

注意：`cameraUtils.js` 中还定义了 `OTS_LEFT` / `OTS_RIGHT`，但当前底部 `Shot Size` 下拉框没有暴露这两个选项。

### 3.4 Camera Angle 选项

关键位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js:261` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:267`

UI 下拉框暴露 5 个机位角度：

| UI label | 存储值 | 实际角度映射 |
| --- | --- | --- |
| Bird's Eye | `ShotAngles.BIRDS_EYE` / `Birds` | `-30°` |
| High | `ShotAngles.HIGH` / `High` | `-15°` |
| Eye | `ShotAngles.EYE` / `Eye` | `0°` |
| Low | `ShotAngles.LOW` / `Low` | `30°` |
| Worm's Eye | `ShotAngles.WORMS_EYE` / `Worms` | `45°` |

角度映射定义在 `src/js/shot-generator/utils/cameraUtils.js:52` 到 `src/js/shot-generator/utils/cameraUtils.js:58`。

### 3.5 下拉框渲染与事件派发

关键位置：

- `src/js/shot-generator/components/CameraPanelInspector/index.js:328` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:341`

`Shot Size` 选择时：

```js
onSetValue={ (item) => onSetShot({ size: item.value, angle: shotInfo.angle }) }
```

`Camera Angle` 选择时：

```js
onSetValue={ (item) => onSetShot({ size: shotInfo.size, angle: item.value }) }
```

这两个回调的设计重点是：选择其中一个维度时保留另一个维度。例如只改 `Shot Size` 时会带上当前 `angle`；只改 `Camera Angle` 时会带上当前 `size`。

通用 `Select` 组件位于 `src/js/shot-generator/components/Select/index.js:27` 到 `src/js/shot-generator/components/Select/index.js:39`，底层使用 `react-select`，`placeholder={ label }`，因此未选择时显示 `Shot Size` 或 `Camera Angle`。

## 4. Redux 状态实现

### 4.1 状态结构

文件：`src/js/shared/reducers/shot-generator.js`

初始状态：

- `src/js/shared/reducers/shot-generator.js:751`

```js
cameraShots: {}
```

单个相机的记录由 `getCameraShot` 初始化：

- `src/js/shared/reducers/shot-generator.js:403` 到 `src/js/shared/reducers/shot-generator.js:412`

```js
{
  size: null,
  angle: null,
  cameraId: cameraId
}
```

实际 state 形态类似：

```js
cameraShots: {
  "camera-uuid": {
    size: "Medium",
    angle: "Eye",
    cameraId: "camera-uuid",
    character: undefined
  }
}
```

### 4.2 Action creator

关键位置：

- `src/js/shared/reducers/shot-generator.js:1757`

```js
setCameraShot: (cameraId, values) => ({
  type: 'SET_CAMERA_SHOT',
  payload: { cameraId, ...values }
})
```

UI 层调用 `setCameraShot(activeCamera.id, { size, angle })` 后，payload 会包含 `cameraId`、`size`、`angle`。

### 4.3 Reducer 更新逻辑

关键位置：

- `src/js/shared/reducers/shot-generator.js:754` 到 `src/js/shared/reducers/shot-generator.js:763`

```js
case 'SET_CAMERA_SHOT':
  const camera = getCameraShot(draft, action.payload.cameraId)

  camera.size = action.payload.size || camera.size
  camera.angle = action.payload.angle || camera.angle
  camera.character = action.payload.character
  return
```

实现特征：

1. `getCameraShot` 保证当前相机有记录。
2. `size` 和 `angle` 使用 `||` 合并，因此新 payload 没有传值时会保留旧值。
3. `character` 每次都会直接赋值，payload 不传时会变成 `undefined`。
4. 当前实现没有清空 `size` / `angle` 的 UI 入口；即使传 `null`，也会因为 `|| camera.size` 被旧值覆盖。

### 4.4 接入全局 reducer

关键位置：

- `src/js/shared/reducers/shot-generator.js:1682` 到 `src/js/shared/reducers/shot-generator.js:1685`

```js
const cameraShots = cameraShotsReducer(state.cameraShots, action)
return (cameraShots !== state.cameraShots) ? { ...state, cameraShots } : state
```

这意味着 `SET_CAMERA_SHOT` 只更新非 undoable 的 `cameraShots` 状态；真正的相机位置变化由后续 `updateObject` 写入 `undoable.sceneObjects`。

## 5. 场景层如何应用自动构图

文件：`src/js/shot-generator/SceneManagerR3fLarge.js`

关键位置：

- `src/js/shot-generator/SceneManagerR3fLarge.js:236` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:258`

该组件监听 `cameraShots`：

```js
useEffect(() => {
  if(!selectedCharacters.current) return
  let selected = scene.children[0].children.find((obj) => selectedCharacters.current.indexOf(obj.userData.id) >= 0)
  let characters = scene.children[0].children.filter((obj) => obj.userData.type === "character")
  if (characters.length) {
    let keys = Object.keys(cameraShots)
    for(let i = 0; i < keys.length; i++ ) {
      let key = keys[i]
      if(cameraShots[key].character) {
        selected = scene.__interaction.filter((object) => object.userData.id === cameraShots[key].character)[0]
      }
      if((!cameraShots[key].size && !cameraShots[key].angle) || camera.userData.id !== cameraShots[key].cameraId ) continue
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

执行条件：

1. 场景中必须有角色。
2. 当前 `cameraShots[key]` 至少有 `size` 或 `angle`。
3. `camera.userData.id` 必须等于 `cameraShots[key].cameraId`，即只对当前 3D camera 对应的记录生效。

目标角色选择逻辑：

1. 默认使用当前选中的角色：`selectedCharacters.current` 中第一个能在场景里找到的角色。
2. 如果 `cameraShots[key].character` 存在，则改用这个 character id 指向的角色。
3. 如果传给 `setShot` 的 `selected` 为空，`setShot` 内部会调用 `getClosestCharacter(characters, camera)` 兜底。

## 6. cameraUtils：Shot Size 的核心算法

文件：`src/js/shot-generator/utils/cameraUtils.js`

### 6.1 Shot Size 枚举

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:29` 到 `src/js/shot-generator/utils/cameraUtils.js:42`

`ShotSizes` 是字符串枚举，UI 和算法都以这些字符串为 key。当前底部面板使用其中 10 个；`OTS_LEFT` / `OTS_RIGHT` 主要供其他自动镜头逻辑扩展。

### 6.2 每种景别对应的骨骼范围

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:60` 到 `src/js/shot-generator/utils/cameraUtils.js:145`

`ShotSizesInfo` 决定某个景别需要把人物的哪些骨骼纳入画面。例如：

- `EXTREME_CLOSE_UP`：`Head`、`leaf`、`Neck`，并对 box 底部做 8cm 裁切。
- `MEDIUM`：头、脖子、肩、`Hips`。
- `LONG`：头、肩、躯干、腿、脚部叶子骨骼。
- `EXTREME_LONG`：同样框入全身，但用 `relativeToScreen: 1 / 3` 让人物高度约占画面高度三分之一。
- `OTS_LEFT` / `OTS_RIGHT`：带 `backSide: true` 和水平 `pan` 偏移，用于越肩镜头，但底部 UI 没有暴露。

### 6.3 getShotBox：根据骨骼生成构图盒

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:230` 到 `src/js/shot-generator/utils/cameraUtils.js:269`

`getShotBox(character, shotType)` 的逻辑：

1. 创建 `THREE.Box3`。
2. 如果 `shotType` 没有对应 `ShotSizesInfo`，只取 `leaf` 骨骼作为目标点。这用于只有 `Camera Angle`、没有 `Shot Size` 的情况。
3. 如果有景别配置，则从角色的 `SkinnedMesh` 中筛选指定骨骼。
4. 遍历骨骼，把每个骨骼的世界坐标起点和终点扩展到 box。
5. 根据 `yReduction` 或 `relativeToScreen` 调整 box 的垂直范围。

这里的 `leaf` 从上下文看是头部/眼睛附近的辅助骨骼；当没有景别时，用它作为默认看向点。

### 6.4 clampCameraToBox：根据 FOV 推导相机距离

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:3` 到 `src/js/shot-generator/utils/cameraUtils.js:19`

`clampCameraToBox` 先把 `Box3` 转为包围球，再用透视相机 FOV 算出能容纳该球所需的相机距离：

```js
let h = sphere.radius / Math.tan(camera.fov / 2 * Math.PI / 180.0)
let newPos = sphere.center + direction * h
```

因此 Shot Size 的本质是：

```text
景别 -> 骨骼集合 -> Box3 -> 包围球 -> 根据 FOV 计算相机距离 -> 相机看向 box 中心
```

### 6.5 getShotInfo：景别整体计算

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:155` 到 `src/js/shot-generator/utils/cameraUtils.js:228`

主要步骤：

1. 从当前相机世界方向得到反向 `direction`，用于保持当前镜头大致朝向。
2. 调用 `getShotBox(selected, shotSize)` 生成目标 box。
3. 如果是 `ESTABLISHING`，把所有角色扩展进 box；多角色时还会根据角色之间的位置关系调整方向。
4. 如果没有有效 `shotSize`，返回当前相机位置和 `leaf` 目标点，只为 Camera Angle 提供旋转基准。
5. 如果景别配置含 `backSide`，反转方向。
6. 调用 `clampCameraToBox` 算出相机位置与目标点。
7. 如果景别配置含 `pan`，额外做水平偏移。

## 7. cameraUtils：Camera Angle 的核心算法

### 7.1 Camera Angle 枚举和角度值

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:44` 到 `src/js/shot-generator/utils/cameraUtils.js:58`

`ShotAnglesInfo` 把字符串枚举映射为弧度：

```js
Birds -> -30°
High -> -15°
Eye -> 0°
Low -> 30°
Worms -> 45°
```

这里的正负并不是最终 UI 看到的“上/下”直觉值，而是传给 `setFromAxisAngle` 前的内部俯仰偏移。实际应用时又使用了负号：`quaternion.setFromAxisAngle(mainAxis, -ShotAnglesInfo[shotAngle])`。

### 7.2 setShot 中的角度应用

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:303` 到 `src/js/shot-generator/utils/cameraUtils.js:354`
- 其中 Camera Angle 逻辑位于 `src/js/shot-generator/utils/cameraUtils.js:319` 到 `src/js/shot-generator/utils/cameraUtils.js:333`

执行步骤：

1. 先调用 `getShotInfo` 得到 `clampedInfo.position`、`clampedInfo.target`、`direction` 和 `box`。
2. 如果 `shotAngle` 在 `ShotAnglesInfo` 中存在，则计算当前相机到目标点的距离。
3. 把 `direction.y = 0`，只保留水平面方向，避免在已有俯仰基础上继续叠加误差。
4. 用 `camera.up` 和水平 `direction` 叉乘得到旋转轴 `mainAxis`。
5. 用 `ShotAnglesInfo[shotAngle]` 创建四元数。
6. 旋转 `direction`，再恢复原距离。
7. 令相机位置等于 `target - direction`。

换句话说，Camera Angle 不直接设置 `tilt`，而是先围绕目标点改变相机位置高度，再调用 `camera.lookAt(target)` 得到最终俯仰。

### 7.3 地面防穿透处理

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:335` 到 `src/js/shot-generator/utils/cameraUtils.js:338`

如果计算后的相机 `position.y < 0`，代码会沿水平投影做一次修正，并把高度固定为 0：

```js
if (clampedInfo.position.y < 0) {
  clampedInfo.position.sub(direction.clone().setY(0).setLength(clampedInfo.position.y))
  clampedInfo.position.y = 0
}
```

这主要影响低角度、虫视角等可能把相机推到地面以下的情况。

### 7.4 写回 Storyboarder 的坐标字段

关键位置：

- `src/js/shot-generator/utils/cameraUtils.js:340` 到 `src/js/shot-generator/utils/cameraUtils.js:352`

`setShot` 先修改 Three.js camera：

```js
camera.position.copy(clampedInfo.position)
camera.lookAt(clampedInfo.target)
camera.updateMatrixWorld(true)
```

然后把 Three.js 坐标转换回 Storyboarder scene object 字段：

```js
updateObject(camera.userData.id, {
  x: camera.position.x,
  y: camera.position.z,
  z: camera.position.y,
  rotation: rot.y,
  roll: rot.z,
  tilt: rot.x
})
```

注意这里存在坐标轴转换：

- Three.js `position.x` -> scene object `x`
- Three.js `position.z` -> scene object `y`
- Three.js `position.y` -> scene object `z`

旋转则从 camera quaternion 转成 `YXZ` 欧拉角后写入 `rotation`、`roll`、`tilt`。

## 8. 只选 Shot Size / 只选 Camera Angle 的行为

### 8.1 只选 Shot Size

当用户只选 `Shot Size`，`shotAngle` 为空：

1. `getShotInfo` 会按景别计算人物包围盒。
2. `clampCameraToBox` 根据当前相机方向和 FOV 计算距离。
3. `setShot` 不进入 `ShotAnglesInfo[shotAngle]` 分支。
4. 最终效果是保持当前大致机位方向，只调整距离和 lookAt，让指定人物部位进入画面。

### 8.2 只选 Camera Angle

当用户只选 `Camera Angle`，`shotSize` 为空：

1. `getShotBox` 会进入 `!ShotSizesInfo[shotType]` 分支，只取 `leaf` 骨骼。
2. `getShotInfo` 返回当前相机位置和 `leaf` 中心作为目标点。
3. `setShot` 根据 `Camera Angle` 围绕该目标点调整相机高度/方向。
4. 最终效果是尽量以人物头部/眼睛附近为目标，仅改变俯仰角度，不重新套用具体景别距离。

### 8.3 同时选择 Shot Size 和 Camera Angle

这是完整自动构图路径：

1. `Shot Size` 决定框入人物哪些部位，以及相机到目标 box 的距离。
2. `Camera Angle` 在该目标和距离基础上，围绕目标点重新摆放相机高度。
3. `camera.lookAt(target)` 得到最终 `rotation` / `tilt` / `roll`。

## 9. 与 Shot Explorer 的关系

文件：`src/js/shot-explorer/ShotMaker.js`

关键位置：

- `src/js/shot-explorer/ShotMaker.js:130` 到 `src/js/shot-explorer/ShotMaker.js:142`

Shot Explorer 生成候选镜头时，也从 `cameraUtils` 导入：

```js
import { ShotSizes, ShotAngles, setShot } from '../shot-generator/utils/cameraUtils'
```

它会随机选择 `ShotAngles` 和 `ShotSizes`，复制当前相机，调用同一个 `setShot` 生成镜头构图。这说明底部 `Shot Size` / `Camera Angle` 与 Shot Explorer 使用同一套核心算法。

## 10. 重要实现细节与潜在问题

1. `SET_CAMERA_SHOT` 使用 `action.payload.size || camera.size` 和 `action.payload.angle || camera.angle`，所以不能通过传 `null` 或空字符串清空已有值。
2. `camera.character = action.payload.character` 每次都会覆盖，UI 面板当前没有传 `character`，因此从面板选择景别或角度后会把已有 `character` 写成 `undefined`。
3. `SceneManagerR3fLarge` 的监听依赖只有 `[cameraShots]`。角色姿态、角色位置或相机 FOV 改变时，如果 `cameraShots` 不变，不一定会重新触发自动构图。
4. `getShotBox` 假设角色存在 `SkinnedMesh` 和指定骨骼；如果模型骨骼命名不同，可能导致 box 为空或构图不准确。
5. `ESTABLISHING` 会把所有角色扩入 box，因此与单角色景别不同，它是场景级/群体级构图。
6. Camera Angle 的实现是改变相机位置再 `lookAt`，不是单纯修改 `tilt`。因此不同当前相机方向、不同目标点会产生不同最终位置。

## 11. 总结

`Shot Size` 和 `Camera Angle` 的实现分为三层：

1. UI 层：`CameraPanelInspector` 渲染两个下拉框，选择后派发 `setCameraShot`。
2. 状态层：`cameraShotsReducer` 按相机 id 保存景别和角度。
3. 算法层：`SceneManagerR3fLarge` 监听状态变化并调用 `cameraUtils.setShot`，由 `Shot Size` 决定人物包围盒与相机距离，由 `Camera Angle` 决定围绕目标点的相机高度/俯仰，最终通过 `updateObject` 写回当前相机对象。

因此，这两个功能的真正核心不在下拉 UI，而在 `src/js/shot-generator/utils/cameraUtils.js` 中的 `ShotSizesInfo`、`ShotAnglesInfo`、`getShotBox`、`getShotInfo` 和 `setShot`。
