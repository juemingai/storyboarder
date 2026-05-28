# Shot Generator 初始页面默认值设计分析

本文分析用户从 Storyboarder 主界面点击 **Shot Generator** 后，进入截图所示 Shot Generator 页面时，页面默认值如何被初始化、加载、渲染，以及这些默认值对应的代码文件和位置。

## 结论摘要

截图中的页面是当前 storyboard board 没有已保存 `board.sg` 数据时的空场景默认状态。代码路径是：

1. 主窗口 Shot Generator 面板点击后发送 `shot-generator:open`。
2. Electron 主进程打开 Shot Generator 窗口，并发送 `shot-generator:reload`。
3. Shot Generator renderer 获取当前 board。
4. 如果当前 board 没有 `board.sg`，调用 `resetScene()`。
5. `resetScene()` 使用 `initialState.undoable` 中的 `initialScene`：
   - 只包含 1 个默认相机。
   - 没有默认人物、物体、灯光对象。
   - 地面网格打开。
   - 房间关闭。
   - 背景色为 `0xE5E5E5`。
   - 环境模型为空。
   - 主视图相机模式重置为 `live`。

需要特别注意：代码里还存在一个 `defaultScenePreset`，它包含 box、chair、character、light、camera 等示例对象，但这不是截图中“第一次进入当前 board 的默认空场景”使用的初始状态。截图状态来自 `initialScene`。

## 点击进入页面的加载链路

| 步骤 | 文件位置 | 说明 |
| --- | --- | --- |
| 主界面渲染 Shot Generator 面板 | `src/js/window/main-window.js:3592` | `renderShotGeneratorPanel()` 读取当前 board 的 Shot Generator 缩略图和项目画幅比例。 |
| 点击面板发送打开事件 | `src/js/window/main-window.js:3608`、`src/js/window/main-window.js:3619` | `onOpen` 触发 `ipcRenderer.send('shot-generator:open')`。 |
| 主进程打开 Shot Generator 窗口 | `src/js/main.js:1699` | 监听 `shot-generator:open`。 |
| 窗口 ready 后请求 reload | `src/js/main.js:1702`、`src/js/main.js:1703` | `shotGeneratorWindow.show(...)` 后向 Shot Generator renderer 发送 `shot-generator:reload`。 |
| 创建 Shot Generator BrowserWindow | `src/js/windows/shot-generator/main.js:9`、`src/js/windows/shot-generator/main.js:29`、`src/js/windows/shot-generator/main.js:93` | 使用 `shotGeneratorSize` 或默认 `1505 x 1080` 窗口尺寸，加载 `shot-generator.html`。 |
| Shot Generator renderer 接收 reload | `src/js/windows/shot-generator/window.js:213` | 进入页面数据初始化逻辑。 |
| 获取当前 storyboarder 文件和当前 board | `src/js/windows/shot-generator/window.js:214`、`src/js/windows/shot-generator/window.js:215` | 读取 `storyboarderFilePath`、`boardData`、当前 `board`。 |
| 设置画幅比例 | `src/js/windows/shot-generator/window.js:216`、`src/js/windows/shot-generator/window.js:222` | 从 `boardData.aspectRatio` 转成数字，dispatch `SET_ASPECT_RATIO`。 |
| 加载当前 board 数据 | `src/js/windows/shot-generator/window.js:235` | 调用 `loadBoard(board)`。 |
| 判断是否有已保存 Shot Generator 数据 | `src/js/shared/actions/load-board-from-data.js:9`、`src/js/shared/actions/load-board-from-data.js:14` | 如果 `board.sg` 存在，加载保存的场景；否则重置为空默认场景。 |
| 无 `board.sg` 时使用默认场景 | `src/js/shared/actions/load-board-from-data.js:17` | dispatch `resetScene()`。 |
| 清空撤销历史 | `src/js/shared/actions/load-board-from-data.js:20` | 进入页面后历史栈干净。 |

## Store 初始化默认值

Shot Generator 窗口启动时会基于 reducer 导出的 `initialState` 创建 Redux store。

| 默认项 | 文件位置 | 默认值 / 作用 |
| --- | --- | --- |
| 创建 store | `src/js/windows/shot-generator/window.js:124` | `configureStore({ ...initialState, presets: ... })`。 |
| 合并用户预设 | `src/js/windows/shot-generator/window.js:126` 到 `src/js/windows/shot-generator/window.js:149` | 将本地保存的 scene / character / pose / hand pose / emotion presets 合并进默认预设。 |
| 预加载人物与 XR 资源 | `src/js/windows/shot-generator/window.js:151` 到 `src/js/windows/shot-generator/window.js:165` | 默认预加载 `adult-male`、`bone.glb`、`light.glb`、`hmd.glb`，用于后续交互，并不代表默认场景中已经有这些对象。 |
| 默认应用状态 | `src/js/shared/reducers/shot-generator.js:614` | 定义 `initialState`。 |
| 默认画幅比例 | `src/js/shared/reducers/shot-generator.js:671` | `aspectRatio: 2.35`，但进入具体 storyboard 项目后会被 `boardData.aspectRatio` 覆盖。 |
| 默认 board | `src/js/shared/reducers/shot-generator.js:673` | 初始为 `{}`；`SET_BOARD` 后才有 `uid`，Editor 才渲染页面。 |
| 默认 undoable 场景数据 | `src/js/shared/reducers/shot-generator.js:675` 到 `src/js/shared/reducers/shot-generator.js:682` | 使用 `initialScene.world`、`initialScene.activeCamera`、`initialScene.sceneObjects`，并清空选择。 |

## 空场景默认值：initialScene

截图对应的核心默认值定义在 `src/js/shared/reducers/shot-generator.js:562` 到 `src/js/shared/reducers/shot-generator.js:611`。

### 世界 / 场景默认值

| 字段 | 文件位置 | 默认值 | 截图表现 |
| --- | --- | --- | --- |
| `world.ground` | `src/js/shared/reducers/shot-generator.js:564` | `true` | 主视图底部显示地面网格；左侧 Scene 面板“地面”开启。 |
| `world.backgroundColor` | `src/js/shared/reducers/shot-generator.js:565` | `0xE5E5E5` | 主视图背景是浅灰色；左侧背景色滑块约为 `0.90`。 |
| `world.room.visible` | `src/js/shared/reducers/shot-generator.js:567` | `false` | 默认不显示房间墙体；左侧“房间 / 可见性”未勾选。 |
| `world.room.width` | `src/js/shared/reducers/shot-generator.js:568` | `10` | 左侧房间宽度默认 `10.00`。 |
| `world.room.length` | `src/js/shared/reducers/shot-generator.js:569` | `10` | 左侧房间长度默认 `10.00`。 |
| `world.room.height` | `src/js/shared/reducers/shot-generator.js:570` | `3` | 左侧房间高度默认 `3.00`。 |
| `world.environment.file` | `src/js/shared/reducers/shot-generator.js:573` | `undefined` | 默认没有环境模型。 |
| `world.environment.x/y/z` | `src/js/shared/reducers/shot-generator.js:574` 到 `src/js/shared/reducers/shot-generator.js:576` | `0, 0, 0` | 环境模型位置默认在原点。 |
| `world.environment.rotation` | `src/js/shared/reducers/shot-generator.js:577` | `0` | 环境模型旋转默认 0。 |
| `world.environment.scale` | `src/js/shared/reducers/shot-generator.js:578` | `1` | 环境模型缩放默认 1。 |
| `world.environment.visible` | `src/js/shared/reducers/shot-generator.js:579` | `true` | 若选择环境文件，默认可见。 |
| `world.environment.grayscale` | `src/js/shared/reducers/shot-generator.js:580` | `true` | 环境模型默认灰度化。 |
| `world.ambient.intensity` | `src/js/shared/reducers/shot-generator.js:582`、`src/js/shared/reducers/shot-generator.js:583` | `0.5` | 默认环境光强度。 |
| `world.directional.intensity` | `src/js/shared/reducers/shot-generator.js:586` | `0.5` | 默认方向光强度。 |
| `world.directional.rotation` | `src/js/shared/reducers/shot-generator.js:587` | `-0.9` 弧度 | 默认方向光水平旋转。 |
| `world.directional.tilt` | `src/js/shared/reducers/shot-generator.js:588` | `0.75` 弧度 | 默认方向光俯仰。 |
| `world.fog.visible` | `src/js/shared/reducers/shot-generator.js:591` | `true` | 默认开启雾。 |
| `world.fog.far` | `src/js/shared/reducers/shot-generator.js:592` | `40` | 雾距离默认 40。 |
| `world.shadingMode` | `src/js/shared/reducers/shot-generator.js:594` | `ShadingType.Outline` | 左侧 Scene 面板显示 `Outline`；主视图使用描边风格。 |

### 默认相机对象

默认场景只有一个相机对象，定义在 `src/js/shared/reducers/shot-generator.js:596` 到 `src/js/shared/reducers/shot-generator.js:610`。

| 字段 | 文件位置 | 默认值 | 截图表现 |
| --- | --- | --- | --- |
| 相机 id | `src/js/shared/reducers/shot-generator.js:597`、`src/js/shared/reducers/shot-generator.js:610` | `6BC46A44-7965-43B5-B290-E3D2B9D15EEE` | 该 id 同时是 `activeCamera`。 |
| `type` | `src/js/shared/reducers/shot-generator.js:599` | `camera` | 左上 top-down 视图和左侧列表出现 `Camera 1`。 |
| `fov` | `src/js/shared/reducers/shot-generator.js:600` | `22.25` | 底部镜头控件换算后显示约 `37.89mm`。 |
| `x` | `src/js/shared/reducers/shot-generator.js:601` | `0` | 相机默认在场景 X=0。 |
| `y` | `src/js/shared/reducers/shot-generator.js:602` | `6` | 业务坐标的 Y 映射到 Three.js Z，表示相机离目标较远。 |
| `z` | `src/js/shared/reducers/shot-generator.js:603` | `1` | 底部提升控件显示 `1.00m`。 |
| `rotation` | `src/js/shared/reducers/shot-generator.js:604` | `0` | 底部转动 / pan 显示 `0°`。 |
| `tilt` | `src/js/shared/reducers/shot-generator.js:605` | `0` | 底部倾斜 / tilt 显示 `0°`。 |
| `roll` | `src/js/shared/reducers/shot-generator.js:606` | `0.0` | 底部 roll 显示 `0°`。 |
| `name` | `src/js/shared/reducers/shot-generator.js:607` | `undefined` | 经过 `withDisplayNames` 后显示为 `Camera 1`。 |

## 为什么截图中默认选中 Scene 而不是 Camera

默认进入空场景时，选择列表是空数组：

- 初始定义：`src/js/shared/reducers/shot-generator.js:679`，`selections: []`。
- `LOAD_SCENE` 时清空选择：`src/js/shared/reducers/shot-generator.js:800` 到 `src/js/shared/reducers/shot-generator.js:807`。

左侧列表的 `Scene` 是一个虚拟列表项，不是 `sceneObjects` 里的对象。当 `selections.length === 0` 时它显示为选中：

- `src/js/shot-generator/components/ItemList/index.js:152` 到 `src/js/shot-generator/components/ItemList/index.js:154`。

属性面板根据当前选择判断显示什么：

- `src/js/shot-generator/components/ElementsPanel/index.js:62`、`src/js/shot-generator/components/ElementsPanel/index.js:63`：通过 `selections[0]` 找当前对象。
- `src/js/shot-generator/components/ElementsPanel/Inspector.js:14` 到 `src/js/shot-generator/components/ElementsPanel/Inspector.js:19`：没有选中对象时渲染 `<InspectedWorld/>`。

因此截图中左侧列表选中 `Scene`，下方显示地面、背景色、阴影模式、房间等世界属性。

## 默认值如何渲染到截图 UI

### 页面布局

Shot Generator 页面主布局由 `Editor` 组合：

| 区域 | 文件位置 | 说明 |
| --- | --- | --- |
| 没有 board uid 时不渲染 | `src/js/shot-generator/components/Editor/index.js:56`、`src/js/shot-generator/components/Editor/index.js:57` | 等待 `SET_BOARD` 设置当前 board 后再显示 UI。 |
| 顶部工具栏 | `src/js/shot-generator/components/Editor/index.js:190` 到 `src/js/shot-generator/components/Editor/index.js:195` | 渲染“相机、物体、人物、灯光、音量、图片、VR、保存到绘板、添加为新板”等按钮。 |
| 左上 top-down 视图 | `src/js/shot-generator/components/Editor/index.js:199` 到 `src/js/shot-generator/components/Editor/index.js:216` | 渲染 `SceneManagerR3fSmall`，显示相机俯视图。 |
| 左侧对象列表 + Inspector | `src/js/shot-generator/components/Editor/index.js:227`、`src/js/shot-generator/components/ElementsPanel/index.js:67` 到 `src/js/shot-generator/components/ElementsPanel/index.js:70` | 显示 `Scene` 和 `Camera 1`，下方为世界 Inspector。 |
| 主相机视图 | `src/js/shot-generator/components/Editor/index.js:233` 到 `src/js/shot-generator/components/Editor/index.js:252` | 渲染 `SceneManagerR3fLarge`。 |
| 底部控制区 | `src/js/shot-generator/components/Editor/index.js:258` 到 `src/js/shot-generator/components/Editor/index.js:264` | 包含相机控制、Shot 信息、相机选择、导览/网格/视图按钮等。 |

### 世界 Inspector

`InspectedWorld` 将 `world` 默认值映射到左侧 Scene 面板：

| UI | 文件位置 | 数据来源 |
| --- | --- | --- |
| Scene 标题 | `src/js/shot-generator/components/InspectedWorld/index.js:89` | 国际化 key `shot-generator.inspector.inspected-world.scene`。 |
| 地面 checkbox | `src/js/shot-generator/components/InspectedWorld/index.js:92` 到 `src/js/shot-generator/components/InspectedWorld/index.js:96` | `world.ground`，默认 `true`。 |
| 背景色滑块 | `src/js/shot-generator/components/InspectedWorld/index.js:100` 到 `src/js/shot-generator/components/InspectedWorld/index.js:107` | `world.backgroundColor / 0xFFFFFF`，`0xE5E5E5` 约等于 `0.90`。 |
| 阴影模式 | `src/js/shot-generator/components/InspectedWorld/index.js:110` 到 `src/js/shot-generator/components/InspectedWorld/index.js:124` | `world.shadingMode`，默认 `Outline`。 |
| 房间可见性 | `src/js/shot-generator/components/InspectedWorld/index.js:130` 到 `src/js/shot-generator/components/InspectedWorld/index.js:135` | `world.room.visible`，默认 `false`。 |
| 房间尺寸 | `src/js/shot-generator/components/InspectedWorld/index.js:137` 到 `src/js/shot-generator/components/InspectedWorld/index.js:140` | `world.room.width/length/height`，默认 `10 / 10 / 3`。 |

### 主 3D 视图

`SceneManagerR3fLarge` 把 world 和 sceneObjects 渲染成 Three.js 场景：

| 渲染项 | 文件位置 | 数据来源 / 说明 |
| --- | --- | --- |
| 背景色 | `src/js/shot-generator/SceneManagerR3fLarge.js:311` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:313` | `scene.background = new THREE.Color(world.backgroundColor)`。 |
| 描边 / shading effect | `src/js/shot-generator/SceneManagerR3fLarge.js:323` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:327` | `mainViewCamera === 'live'` 时使用 `world.shadingMode`。 |
| 环境光 | `src/js/shot-generator/SceneManagerR3fLarge.js:348` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:352` | `world.ambient.intensity`，默认 `0.5`。 |
| 方向光 | `src/js/shot-generator/SceneManagerR3fLarge.js:354` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:360` | `world.directional.intensity` 等。 |
| 方向光旋转 | `src/js/shot-generator/SceneManagerR3fLarge.js:303` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:309` | 用 `world.directional.rotation` 和 `world.directional.tilt` 设置光源角度。 |
| 地面网格 | `src/js/shot-generator/SceneManagerR3fLarge.js:481` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:485` | 只有 `!world.room.visible && world.ground` 时显示，因此空场景显示地面网格。 |
| 环境模型 | `src/js/shot-generator/SceneManagerR3fLarge.js:487` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:495` | 只有 `world.environment.file` 存在时渲染，默认不显示。 |
| 房间 | `src/js/shot-generator/SceneManagerR3fLarge.js:497` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:503` | 组件存在，但 `visible={world.room.visible}`，默认不可见。 |

### 默认相机视角

默认 active camera 来自 `initialScene.activeCamera`。

| 逻辑 | 文件位置 | 说明 |
| --- | --- | --- |
| 读取 activeCamera | `src/js/shot-generator/CameraUpdate.js:15` 到 `src/js/shot-generator/CameraUpdate.js:18` | 从 Redux 中读取当前 active camera 对象。 |
| 坐标映射 | `src/js/shot-generator/CameraUpdate.js:21` 到 `src/js/shot-generator/CameraUpdate.js:23` | Shot Generator 业务坐标映射到 Three.js：`camera.position.x = x`、`camera.position.y = z`、`camera.position.z = y`。 |
| 旋转映射 | `src/js/shot-generator/CameraUpdate.js:24` 到 `src/js/shot-generator/CameraUpdate.js:28` | `rotation` -> yaw，`tilt` -> X 轴旋转，`roll` -> Z 轴旋转。 |
| FOV 应用 | `src/js/shot-generator/CameraUpdate.js:34` 到 `src/js/shot-generator/CameraUpdate.js:37` | `camera.fov = activeCamera.fov` 并更新投影矩阵。 |

默认相机 `x=0, y=6, z=1, rotation=0, tilt=0, roll=0, fov=22.25`，所以主视图是从 1 米高、沿默认方向观察场景，看到浅灰背景和下方地面网格。

### 底部相机控制面板默认值

`CameraPanelInspector` 根据 active camera 和 `cameraShots` 显示底部控件：

| UI | 文件位置 | 默认值来源 |
| --- | --- | --- |
| 当前 active camera | `src/js/shot-generator/components/CameraPanelInspector/index.js:65` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:74` | 来自 `getSceneObjects(state)[getActiveCamera(state)]`。 |
| Shot Size / Camera Angle | `src/js/shot-generator/components/CameraPanelInspector/index.js:76` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:78` | `cameraShots[activeCamera.id] || {}`，默认没有 size / angle，所以 Select 显示 placeholder。 |
| Roll | `src/js/shot-generator/components/CameraPanelInspector/index.js:105` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:113`、`src/js/shot-generator/components/CameraPanelInspector/index.js:276` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:284` | 默认 `activeCamera.roll = 0`，显示 `0°`。 |
| Pan / Tilt | `src/js/shot-generator/components/CameraPanelInspector/index.js:116` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:119`、`src/js/shot-generator/components/CameraPanelInspector/index.js:286` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:292` | 默认 `rotation = 0`、`tilt = 0`，显示 `0° // 0°`。 |
| Elevate | `src/js/shot-generator/components/CameraPanelInspector/index.js:307` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:316` | 默认 `activeCamera.z = 1`，显示 `1.00m`。 |
| Lens | `src/js/shot-generator/components/CameraPanelInspector/index.js:153` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:157`、`src/js/shot-generator/components/CameraPanelInspector/index.js:318` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:326` | 使用 Three.js `PerspectiveCamera.getFocalLength()` 从 `fov=22.25` 换算，截图显示约 `37.89mm`。 |
| Shot Size Select | `src/js/shot-generator/components/CameraPanelInspector/index.js:328` 到 `src/js/shot-generator/components/CameraPanelInspector/index.js:334` | 默认未设置，显示 `Shot Size` 占位。 |
| Camera Angle Select | `src/js/shot-generator/components/CameraPanelInspector/index.js:336` 起 | 默认未设置，显示 `Camera Angle` 占位。 |

### 底部 Camera 选择

默认场景只有一个 camera，所以底部相机选择区只显示 `1`：

- 相机列表来自 `src/js/shot-generator/components/CamerasInspector/index.js:13` 到 `src/js/shot-generator/components/CamerasInspector/index.js:20`，筛选 `sceneObjects` 中 `type === "camera"` 的对象。
- 点击相机编号会同时 `selectObject(camera.id)` 和 `setActiveCamera(camera.id)`：`src/js/shot-generator/components/CamerasInspector/index.js:100` 到 `src/js/shot-generator/components/CamerasInspector/index.js:106`。
- 第一个相机按钮显示为 `1`：`src/js/shot-generator/components/CamerasInspector/index.js:109` 到 `src/js/shot-generator/components/CamerasInspector/index.js:117`。

### Board 信息

截图底部显示 `Shot 1A`，来自当前 storyboard board 的元数据：

- `SET_BOARD` 将当前 board 的 `uid`、`shot`、`dialogue`、`action`、`notes` 写入 store：`src/js/shared/reducers/shot-generator.js:1519` 到 `src/js/shared/reducers/shot-generator.js:1529`。
- `BoardInspector` 显示 `Shot ` + `board.shot`：`src/js/shot-generator/components/BoardInspector/index.js:12` 到 `src/js/shot-generator/components/BoardInspector/index.js:15`。

## 有已保存 Shot Generator 数据时不是这些默认值

如果当前 board 已经保存过 Shot Generator 数据，即 `board.sg` 存在，加载逻辑会走另一条路径：

- `src/js/shared/actions/load-board-from-data.js:14`、`src/js/shared/actions/load-board-from-data.js:15`：dispatch `loadScene(shot.data)`。
- `src/js/shared/reducers/shot-generator.js:898` 到 `src/js/shared/reducers/shot-generator.js:906`：`sceneObjectsReducer` 使用保存的 `action.payload.sceneObjects`。
- `src/js/shared/reducers/shot-generator.js:1257` 到 `src/js/shared/reducers/shot-generator.js:1259`：`activeCameraReducer` 使用保存的 `action.payload.activeCamera`。
- `src/js/shared/reducers/shot-generator.js:1299` 到 `src/js/shared/reducers/shot-generator.js:1306`：`worldReducer` 使用保存的 `action.payload.world`，同时执行旧数据迁移。

因此，本文列出的默认值只适用于“当前 board 没有已保存 `board.sg` 数据”的首次进入状态。

## `defaultScenePreset` 与 `initialScene` 的区别

`src/js/shared/reducers/shot-generator.js:433` 到 `src/js/shared/reducers/shot-generator.js:560` 定义了 `defaultScenePreset`。它包含：

- `world.ground: false`
- `world.room.visible: true`
- 3 个 object
- 1 个 character
- 1 个 light
- 1 个 camera

这更像一个“场景预设 / 示例预设”的数据结构，不是截图中的默认空场景。实际首次进入当前 board 时，`resetScene()` 取的是：

- `src/js/shared/reducers/shot-generator.js:1779` 到 `src/js/shared/reducers/shot-generator.js:1785`
- 其中 `world`、`sceneObjects`、`activeCamera` 分别来自 `initialState.undoable`
- 而 `initialState.undoable` 又来自 `initialScene`：`src/js/shared/reducers/shot-generator.js:675` 到 `src/js/shared/reducers/shot-generator.js:679`

## 默认值一览

| 分类 | 默认值 |
| --- | --- |
| 主视图模式 | `mainViewCamera: 'live'`，`LOAD_SCENE` 时重置，见 `src/js/shared/reducers/shot-generator.js:1456` 到 `src/js/shared/reducers/shot-generator.js:1458`。 |
| 画幅比例 | Store 初始 `2.35`，进入项目后由 `boardData.aspectRatio` 覆盖，见 `src/js/shared/reducers/shot-generator.js:671`、`src/js/windows/shot-generator/window.js:216` 到 `src/js/windows/shot-generator/window.js:224`。 |
| 选择状态 | `[]`，因此选中虚拟 `Scene` 项，见 `src/js/shared/reducers/shot-generator.js:679`、`src/js/shared/reducers/shot-generator.js:800` 到 `src/js/shared/reducers/shot-generator.js:807`。 |
| 世界背景 | `0xE5E5E5`，Inspector 显示约 `0.90`。 |
| 地面 | 开启。 |
| 房间 | 关闭，但尺寸默认 `10 x 10 x 3`。 |
| 环境模型 | 无文件，默认不渲染。 |
| 光照 | 环境光 `0.5`，方向光 `0.5`，方向光旋转 `-0.9`，倾斜 `0.75`。 |
| 雾 | 开启，`far: 40`。 |
| 渲染风格 | `Outline`。 |
| 场景对象 | 只有一个默认相机。 |
| 默认相机 | id `6BC46A44-7965-43B5-B290-E3D2B9D15EEE`，`fov: 22.25`，`x: 0`，`y: 6`，`z: 1`，`rotation/tilt/roll: 0`。 |
| Shot Size / Camera Angle | 默认未设置。 |
