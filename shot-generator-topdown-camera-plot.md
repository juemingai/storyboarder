# Shot Generator 左上角相机预览框 PRD

## 1. 产品背景

Shot Generator 用于搭建 3D 分镜场景、摆放人物、物体、灯光和相机，并生成故事板镜头。用户在主视图中看到的是当前激活相机的取景画面，但在复杂场景中，仅靠主视图难以快速判断：

- 多个相机之间的位置关系
- 人物、物体、灯光在场景中的平面分布
- 当前激活相机的方向、焦距和高度
- 新增相机后是否与原相机或角色重叠
- 需要从哪个相机切换到哪个镜头

因此，Shot Generator 页面左上角提供一个小型俯视预览框，用作场景地图和相机布置图。

## 2. 功能定位

左上角预览框是 Shot Generator 的辅助导航与构图管理工具，定位为：

- 场景俯视图：从顶部观察整个 3D 场景。
- 相机布置图：显示所有相机的位置、朝向、视锥和镜头信息。
- 对象选择入口：用户可以在预览框中点击对象以选中并编辑。
- 相机切换入口：用户点击某个相机即可将其设为当前激活相机。
- 保存分镜的 plot 图来源：保存或插入分镜时，该视图会生成对应的 camera plot 缩略图。

## 3. 目标用户

- 分镜师：快速搭建镜头机位和人物站位。
- 导演/预演用户：理解空间调度和相机关系。
- 动画/影视前期人员：在多个镜头间切换、调整机位和构图。
- VR/远程协作用户：看到远程设备或客户端在场景中的位置反馈。

## 4. 用户问题

- 只能看到当前相机画面，不清楚相机在空间中的实际位置。
- 多相机情况下，不容易知道当前画面来自哪台相机。
- 拖动人物、灯光、相机时缺少全局空间参照。
- 保存到故事板时，除最终镜头图外，还需要一张解释机位关系的图。
- 小场景或大场景都需要自动适配，而不是手动缩放寻找对象。

## 5. 产品目标

- 在左侧区域展示一个固定高度的俯视预览框。
- 自动将场景中所有可见关键对象适配到预览框中。
- 用清晰图标区分相机、人物、物体、灯光、图片、体积对象。
- 对相机显示名称、焦距、离地高度和视锥方向。
- 支持点击选择对象，点击相机时同步切换当前工作相机。
- 支持在俯视图中拖拽对象，快速调整平面位置。
- 支持与主画面互换视图，方便用户放大查看俯视图。
- 在保存或插入分镜时输出 camera plot 图。

## 6. 核心场景

### 6.1 查看当前镜头空间关系

用户进入 Shot Generator 后，主画面显示当前相机的取景画面。左上角预览框展示当前场景中的人物、相机、相机朝向、视锥范围、相机名称和镜头信息，例如 `Camera 2`、`38mm, 1.00m`。用户可以快速判断镜头从哪个方向拍摄人物。

### 6.2 多相机切换

用户创建多个相机后，左上角预览框同时显示所有相机。用户点击 `Camera 2` 后：

- `Camera 2` 被选中。
- `Camera 2` 变为当前激活相机。
- 主视图切换为 `Camera 2` 的取景画面。
- 左侧对象列表中 `Camera 2` 高亮，并显示 active 状态。

### 6.3 拖拽调整位置

用户在俯视图中按下某个对象图标并拖动：

- 对象被选中。
- 对象跟随鼠标在 X/Y 平面移动。
- Redux 场景状态实时更新。
- 主视图同步反映位置变化。
- 若对象被锁定或 blocked，则不能被拖拽。

### 6.4 保存到故事板

用户点击“保存到绘板”或“添加为新版”后：

- 系统保存当前相机画面作为镜头图。
- 同时从左上角俯视图渲染 camera plot 图。
- 两张图一并发送给主进程保存到 storyboarder 数据中。
- 保存时会临时取消选择，避免选中高亮污染最终输出。

### 6.5 放大查看俯视图

用户点击左上角预览框右下角的展开按钮后：

- 小预览框与主视图进行切换。
- 俯视图可在主区域中查看。
- 原主相机画面进入左上角小框。
- 再次点击可切回。

## 7. 当前实现分析

### 7.1 页面结构

主要实现位置：

- `src/js/shot-generator/components/Editor/index.js`
- `src/js/shot-generator/SceneManagerR3fSmall.js`
- `src/js/shot-generator/components/Three/Icons/CameraIcon.js`
- `src/js/shot-generator/hooks/use-save-to-storyboarder.js`

页面布局中，左侧栏 `#aside` 包含两个区域：

- `#topdown`：左上角俯视预览框。
- `#elements`：场景对象列表。

`#topdown` 内部使用 React Three Fiber 的 `Canvas` 渲染小型 Three.js 场景。

### 7.2 渲染模式

左上角预览框使用正交相机：

- `orthographic={true}`
- 相机位置设置在场景上方。
- 相机朝下看。
- 形成俯视地图效果。

当前逻辑：

- 俯视相机 `camera.position.y = 900`
- 俯视相机 `camera.rotation.x = -Math.PI / 2`
- 背景色为白色：`scene.background = new THREE.Color('#FFFFFF')`
- 使用 `useShadingEffect` 进行渲染。
- 当主视图为 live 时，小框显示俯视图。
- 当主视图切换为俯视图时，小框显示原主相机画面。

### 7.3 自动适配视野

`SceneManagerR3fSmall.js` 中的 `autofitOrtho` 会遍历场景关键对象，计算 X/Z 平面最小最大范围，然后自动调整正交相机范围，让所有对象进入预览框。

纳入计算的对象类型：

- `object`
- `character`
- `light`
- `volume`
- `image`
- `camera`

适配规则：

- 如果场景中只有一个对象，额外增加 padding。
- 所有情况下再增加固定 padding。
- 根据预览框或主视图比例修正宽高比。
- 最终设置正交相机的 `left/right/top/bottom`。
- 更新 projection matrix。

### 7.4 对象展示

左上角预览框会显示不同类型对象：

- 相机：使用 `CameraIcon`
- 人物、灯光、体积、图片：使用 `IconsComponent`
- 模型物体：使用 `ModelObject` 的 icon 模式
- 房间：使用 `Room`，并传入 `isTopDown={true}`
- 远程客户端/XR 客户端：通过 `RemoteClients` 和 `XRClient` 渲染

图标文字逻辑：

- 优先显示对象 `name`
- 若无 `name`，显示系统生成的 `displayName`
- 示例：`Camera 1`、`Adult-male 1`

### 7.5 相机图标展示

相机图标显示以下信息：

- 相机图标
- 相机名称，例如 `Camera 1`
- 焦距信息，例如 `38mm`
- 高度信息，例如 `1.00m`
- 左右视锥线，表示相机水平视野范围
- 当前选中状态，图标颜色变为洋红色/粉色

焦距使用 Three.js `PerspectiveCamera` 根据相机 `fov` 计算 `getFocalLength()`，并以 mm 单位显示。高度来自相机对象的 `z`，格式化为两位小数并以 m 为单位显示。

### 7.6 点击与选择

预览框通过 DOM pointer 事件和 Three.js Raycaster 实现点击选择。

点击对象时：

- 系统根据鼠标位置生成 raycaster。
- 检测被点中的 Three.js 对象。
- 找到带 `userData.id` 的上层对象。
- 如果对象未锁定、未 blocked，则选中该对象。
- 如果对象类型是 `camera`，同时调用 `setActiveCamera(id)`。

产品行为：

- 点击普通对象：选中对象，属性面板切换到该对象。
- 点击相机对象：选中相机，并把它设置为当前激活相机。
- 点击空白/地面：取消选择。
- 点击 locked/blocked 对象：不响应选择或拖拽。

### 7.7 拖拽移动

拖拽流程：

1. `pointerdown` 命中对象。
2. 记录当前拖拽对象。
3. 调用 `prepareDrag`。
4. `pointermove` 时调用 `drag`。
5. 调用 `updateStore(updateObjects)` 同步 Redux 状态。
6. `pointerup` 时调用 `endDrag(updateObjects)`。
7. 清空拖拽对象。

产品行为：

- 用户可以在俯视图中拖动物体、人物、灯光、图片、体积对象和相机。
- 拖拽主要影响平面位置。
- 主视图实时更新。
- 被锁定的对象不可拖动。
- 被 blocked 的对象不可拖动。

### 7.8 与左侧对象列表联动

左侧对象列表和俯视预览框共享 Redux 状态：

- 俯视图点击对象后，列表同步高亮。
- 列表点击相机后，俯视图对应相机高亮。
- 列表中相机被选中时，也会 `setActiveCamera`。
- 当前激活相机有 active 图标标识。
- 激活相机不允许删除。

### 7.9 与主画面互换

通过 `mainViewCamera` 控制主视图和小视图显示内容：

- `mainViewCamera === "live"`：主视图显示当前激活相机画面，小框显示俯视图。
- 非 live 状态：主视图显示俯视图，小框显示原主相机画面。

左上角展开按钮调用 `onSwapCameraViewsClick`，用于切换这两个状态。

### 7.10 保存输出

保存或插入分镜时会输出两类图：

- `camera`：主相机最终画面。
- `plot`：左上角俯视 camera plot 图。

示例结构：

```js
images: {
  camera: shotImageDataUrl,
  plot: cameraPlotImageDataUrl
}
```

这说明左上角预览框不仅是 UI 辅助功能，也参与分镜文件的正式输出。

## 8. 功能需求

### 8.1 展示需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-001 | 左上角固定展示场景俯视预览框 | P0 |
| FR-002 | 默认尺寸为左栏宽度 x 300px 高度 | P0 |
| FR-003 | 背景为浅色，保证黑色线框和图标可读 | P0 |
| FR-004 | 自动展示所有关键场景对象 | P0 |
| FR-005 | 自动适配视野，保证对象整体可见 | P0 |
| FR-006 | 支持显示房间边界/空间参考 | P1 |
| FR-007 | 支持显示远程/XR 客户端位置 | P2 |

### 8.2 相机展示需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-101 | 每个相机以相机图标显示 | P0 |
| FR-102 | 相机旁显示相机名称 | P0 |
| FR-103 | 相机旁显示焦距信息 | P0 |
| FR-104 | 相机旁显示高度信息 | P0 |
| FR-105 | 相机显示左右视锥边界 | P0 |
| FR-106 | 选中相机时相机图标高亮 | P0 |
| FR-107 | 相机旋转、FOV、位置变化时，图标和视锥实时同步 | P0 |
| FR-108 | 多相机情况下所有相机同时可见 | P0 |

### 8.3 对象展示需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-201 | 人物对象显示人物图标和名称 | P0 |
| FR-202 | 灯光对象显示灯光图标和名称 | P0 |
| FR-203 | 物体对象显示俯视图模型或图标 | P0 |
| FR-204 | 图片对象显示图片图标和名称 | P1 |
| FR-205 | 体积对象显示体积图标和名称 | P1 |
| FR-206 | 隐藏对象不在俯视图中显示 | P0 |
| FR-207 | 被选中的对象需要高亮 | P0 |

### 8.4 交互需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-301 | 点击对象后选中该对象 | P0 |
| FR-302 | 点击相机后选中该相机 | P0 |
| FR-303 | 点击相机后将其设为当前激活相机 | P0 |
| FR-304 | 点击空白区域取消选择 | P0 |
| FR-305 | 支持拖拽对象调整平面位置 | P0 |
| FR-306 | 拖拽时主视图实时同步更新 | P0 |
| FR-307 | locked 对象不可选择/拖拽 | P0 |
| FR-308 | blocked 对象不可选择/拖拽 | P0 |
| FR-309 | 支持 Shift 多选逻辑与对象列表保持一致 | P2 |
| FR-310 | 选中对象后属性面板显示对应对象属性 | P0 |

### 8.5 视图切换需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-401 | 预览框右下角显示展开/切换按钮 | P0 |
| FR-402 | 点击按钮后，小预览框与主视图互换 | P0 |
| FR-403 | 主视图显示俯视图时，俯视图仍支持观察完整场景 | P0 |
| FR-404 | 切换后渲染比例和相机适配正确 | P0 |

### 8.6 保存输出需求

| 编号 | 需求 | 优先级 |
| --- | --- | --- |
| FR-501 | 保存当前镜头时生成 camera 图 | P0 |
| FR-502 | 保存当前镜头时生成 plot 图 | P0 |
| FR-503 | plot 图应来源于俯视预览场景 | P0 |
| FR-504 | 保存前取消选择，避免高亮状态进入导出图 | P0 |
| FR-505 | plot 图应保持正方形或指定尺寸比例稳定 | P1 |

## 9. 数据需求

场景对象需要至少具备以下字段：

```ts
type SceneObject = {
  id: string
  type: 'camera' | 'character' | 'object' | 'light' | 'volume' | 'image' | 'group'
  x: number
  y: number
  z: number
  rotation?: number | { x: number; y: number; z: number }
  visible?: boolean
  locked?: boolean
  blocked?: boolean
  name?: string
  displayName?: string
}
```

相机对象额外字段：

```ts
type CameraSceneObject = SceneObject & {
  type: 'camera'
  fov: number
  tilt: number
  roll: number
  rotation: number
}
```

全局状态依赖：

```ts
type ShotGeneratorState = {
  sceneObjects: Record<string, SceneObject>
  selections: string[]
  activeCamera: string
  world: {
    room: {
      visible: boolean
      width: number
      length: number
      height: number
    }
    shadingMode: number
    backgroundColor: number
    ambient: {
      intensity: number
    }
    directional: {
      intensity: number
      rotation: number
      tilt: number
    }
  }
  mainViewCamera: 'live' | string
}
```

## 10. 状态流转

### 10.1 点击相机

```text
用户点击 Camera 2
  ↓
Raycaster 命中相机对象
  ↓
读取 userData.id
  ↓
selectObject(Camera 2)
  ↓
setActiveCamera(Camera 2)
  ↓
Redux 更新 selections / activeCamera
  ↓
主视图切换到 Camera 2 画面
  ↓
左侧列表和俯视图同步高亮
```

### 10.2 拖拽对象

```text
pointerdown
  ↓
命中对象
  ↓
prepareDrag
  ↓
pointermove
  ↓
drag 计算新位置
  ↓
updateObjects 写入 Redux
  ↓
主视图/俯视图重新渲染
  ↓
pointerup
  ↓
endDrag
```

### 10.3 保存分镜

```text
点击保存到绘板
  ↓
dispatch selectObject(null)
  ↓
渲染主相机画面
  ↓
渲染俯视 plot 画面
  ↓
生成 dataURL
  ↓
ipcRenderer.send('saveShot')
  ↓
主进程保存 camera / plot 两张图
```

## 11. UI/视觉要求

### 11.1 整体布局

- 位于 Shot Generator 左侧栏顶部。
- 宽度跟随左侧栏，当前为 300px。
- 高度固定 300px。
- 下方紧接场景对象列表。

### 11.2 视觉风格

- 背景浅色，当前为白色。
- 图标和轮廓线需有高对比度。
- 相机视锥线应明显但不喧宾夺主。
- 选中状态使用强识别颜色，当前为洋红色。
- 右下角展开按钮常驻，但不遮挡重要内容。

### 11.3 文字信息

相机文字应包含：

```text
Camera 2
38mm, 1.00m
```

普通对象文字应包含对象名称或用户自定义名称，例如：

```text
Adult-male 1
```

## 12. 交互细节

### 12.1 点击命中规则

- 优先选择距离点击点更近的对象。
- 多对象重叠时，使用渲染顺序辅助判断。
- 点击距离对象过远时不应误选。
- 当前实现中，当点击点与对象平面距离小于约 `0.30` 时认为有效。

### 12.2 锁定规则

- locked 对象不可拖拽，当前实现中点击时也会被忽略。
- blocked 对象不可拖拽，也不可选中。
- invisible 对象不在预览图中展示。

### 12.3 相机激活规则

- 点击相机图标即激活相机。
- 左侧列表点击相机也激活相机。
- 激活相机不能被删除。
- 当前激活相机用于主视图渲染和保存输出。

## 13. 边界情况

| 情况 | 预期行为 |
| --- | --- |
| 场景只有一个相机 | 预览框自动加 padding，避免图标贴边 |
| 场景对象距离很远 | 自动缩放以容纳所有关键对象 |
| 相机被隐藏 | 相机图标不显示 |
| 对象被锁定 | 不允许拖拽 |
| 点击空白区域 | 清空选择 |
| 多相机重叠 | 优先选择更靠近点击位置或渲染顺序靠上的对象 |
| 字体资源未加载 | 图标可加载，文字可能延迟出现 |
| 主视图与小视图切换 | 两边渲染内容互换，不丢失状态 |
| 保存时有选中对象 | 保存前取消选择，输出图不带选中高亮 |

## 14. 非功能需求

### 14.1 性能

- 预览框应保持流畅，拖拽时无明显卡顿。
- 对象数量较多时，应避免重复加载字体和图标资源。
- 自动适配应只在场景对象、比例或字体加载完成后触发。
- 俯视图渲染尺寸保持较小，降低 GPU 压力。

### 14.2 稳定性

- 不应因某个对象缺少 `name`、`displayName` 或 `visible` 字段导致崩溃。
- 场景对象被删除后，预览框应自动移除对应图标。
- activeCamera 丢失时应有降级处理，避免主视图崩溃。

### 14.3 可维护性

- 俯视图渲染逻辑应独立于主视图。
- 相机图标逻辑应封装在 `CameraIcon`。
- 普通对象图标逻辑应封装在 `IconsComponent`。
- 保存输出逻辑应与 UI 操作解耦。

## 15. 验收标准

### 15.1 基础展示

- 打开 Shot Generator 后，左上角出现 300px 高度的俯视预览框。
- 默认场景中至少显示当前相机图标。
- 创建人物后，预览框出现人物图标和名称。
- 创建第二个相机后，预览框显示两个相机。

### 15.2 相机信息

- 每个相机显示名称。
- 每个相机显示焦距 mm。
- 每个相机显示高度 m。
- 修改相机 FOV 后，焦距文本和视锥范围同步变化。
- 修改相机位置后，图标位置同步变化。
- 修改相机旋转后，视锥方向同步变化。

### 15.3 选择与激活

- 点击人物图标，人物被选中。
- 点击相机图标，相机被选中。
- 点击非当前相机后，主视图切换到该相机画面。
- 左侧对象列表中对应对象同步高亮。
- 点击空白区域后取消选择。

### 15.4 拖拽

- 拖拽人物图标，人物平面位置变化。
- 拖拽相机图标，相机位置变化，主视图画面同步变化。
- 锁定对象后，在预览框中无法拖拽该对象。
- 隐藏对象后，该对象从预览框消失。

### 15.5 视图切换

- 点击右下角展开按钮后，俯视图进入主区域。
- 再次点击后，恢复主相机画面在主区域。
- 切换过程中不丢失选中对象、activeCamera 或场景状态。

### 15.6 保存输出

- 点击保存到绘板后，保存数据中包含 `camera` 图。
- 点击保存到绘板后，保存数据中包含 `plot` 图。
- `plot` 图内容与左上角俯视预览一致。
- 保存后的 plot 图不包含选中高亮。

## 16. 当前实现存在的产品风险与可优化点

- 俯视图的点击逻辑和 React Three Fiber 的 `noEvents` 组合较特殊，当前通过 DOM event + raycaster 手动实现，后续维护成本偏高。
- 相机焦距显示有两种格式逻辑：初始化时可能是 `37.89mm`，更新时可能变为 `38mm`，建议统一格式。
- locked 对象在俯视图中当前点击即被忽略，这可能导致用户无法通过俯视图选中 locked 对象查看属性；是否允许“可选中但不可拖拽”需要产品确认。
- 当前自动适配会随着拖拽重新调整视野，可能造成拖拽时画面轻微跳动；可以考虑拖拽期间暂停 autofit。
- 俯视图没有显式缩放/平移控件，复杂大场景中用户只能依赖自动适配。
- 小图文字可能在对象密集时重叠，后续可增加智能避让或悬浮提示。
- 相机图标颜色选中态为洋红色，和当前整体 UI 有较强视觉冲突；如果要现代化设计，可统一到产品色体系。
- 预览框没有网格/坐标轴，空间尺度感依赖对象和房间边界，建议增加可选网格。

## 17. 后续迭代建议

1. 增加手动缩放和平移：支持鼠标滚轮缩放、空格/中键拖动画布。
2. 优化相机标签避让：对象密集时自动偏移标签，避免文字互相遮挡。
3. 增加网格与比例尺：帮助用户理解空间距离。
4. 区分“选中”和“激活相机”：相机被选中不一定立即激活，可通过双击激活；该改动会改变当前工作流，需要谨慎。
5. 增加 Hover Tooltip：鼠标悬停显示完整对象信息，如名称、类型、坐标、锁定状态。
6. 增加视野安全框：在俯视图中显示当前画幅比例和镜头覆盖范围。
7. 优化 locked 行为：允许选中 locked 对象查看属性，但禁止拖拽和编辑位置。
8. 导出 plot 时提供更干净的样式：隐藏控制按钮，使用统一线宽和字体大小。

## 18. 涉及代码清单

### 18.1 页面容器与视图编排

#### `src/js/shot-generator/components/Editor/index.js`

职责：负责 Shot Generator 主页面布局，并挂载左上角俯视预览框。

关键代码点：

- `#aside`：左侧栏容器。
- `#topdown`：左上角俯视预览区域。
- `Canvas id="top-down-canvas"`：承载左上角 Three.js/R3F 小画布。
- `SceneManagerR3fSmall`：实际渲染俯视场景的组件。
- `topdown__controls`：右下角展开/切换按钮容器。
- `onSwapCameraViewsClick`：触发主视图与小视图互换。
- `smallCanvasData`：保存小画布的 `camera`、`scene`、`gl` 引用，用于保存 plot 图。
- `largeCanvasData`：保存主画布的 `camera`、`scene`、`gl` 引用，用于保存最终镜头图。

相关结构：

```jsx
<div id="topdown">
  <Canvas
    key="top-down-canvas"
    id="top-down-canvas"
    tabIndex={0}
    gl2={true}
    orthographic={true}
    updateDefaultCamera={false}
    noEvents={true}
  >
    <Provider store={store}>
      <SceneManagerR3fSmall
        renderData={mainViewCamera === "live" ? null : largeCanvasData.current}
        mainRenderData={mainViewCamera === "live" ? largeCanvasData.current : smallCanvasData.current}
        setSmallCanvasData={setSmallCanvasData}
        mainViewCamera={mainViewCamera}
      />
    </Provider>
  </Canvas>

  <div className="topdown__controls">
    <div className="row" />
    <div className="row">
      <a href="#" onClick={onSwapCameraViewsClick}>
        <Icon src="icon-camera-view-expand" />
      </a>
    </div>
  </div>
</div>
```

产品含义：

- 该文件决定左上角预览框出现在页面中的位置。
- 通过 `mainViewCamera` 决定小框显示俯视图还是主相机图。
- 通过 `setSmallCanvasData` 将小画布信息暴露给保存逻辑。

### 18.2 俯视图场景管理

#### `src/js/shot-generator/SceneManagerR3fSmall.js`

职责：左上角相机预览框的核心实现文件。负责俯视相机、场景对象渲染、点击选择、拖拽、自动适配、相机切换和渲染输出。

关键依赖：

```js
import {
  getSceneObjects,
  getWorld,
  selectObject,
  getSelections,
  updateObjects,
  setActiveCamera
} from '../shared/reducers/shot-generator'
```

这些 Redux selector/action 决定了俯视图和全局场景状态的联动。

关键状态：

- `sceneObjects`：当前场景所有对象。
- `world`：房间、灯光、背景、shading 等世界状态。
- `selections`：当前选中的对象 ID 列表。
- `mainViewCamera`：决定小画布渲染俯视图还是主视图。
- `renderData`：当小画布需要显示主视图时使用的外部渲染数据。
- `mainRenderData`：用于相机 icon 同步主相机实时状态。

#### 18.2.1 对象分类

```js
const modelObjectIds = useMemo(() => {
  return Object.values(sceneObjects)
    .filter(o => o.type === 'object')
    .map(o => o.id)
}, [sceneObjectLength])

const camerasIds = useMemo(() => {
  return Object.values(sceneObjects)
    .filter(o => o.type === 'camera')
    .map(o => o.id)
}, [sceneObjectLength])

const iconIds = useMemo(() => {
  return Object.values(sceneObjects)
    .filter(o =>
      o.type === 'light' ||
      o.type === 'character' ||
      o.type === 'volume' ||
      o.type === 'image'
    )
    .map(o => o.id)
}, [sceneObjectLength])
```

产品含义：

- 相机、模型物体、普通图标对象分开渲染。
- 相机使用独立 `CameraIcon`，因为它需要显示视锥和焦距。
- 人物、灯光、体积、图片使用统一 `IconsComponent`。

#### 18.2.2 俯视相机初始化

```js
useEffect(() => {
  camera.position.y = 900
  camera.rotation.x = -Math.PI / 2
  camera.layers.enable(2)
  camera.updateMatrixWorld(true)
}, [])
```

产品含义：

- 将小画布相机放到场景正上方。
- 向下俯视整个 3D 场景。
- 使用正交相机形成二维地图效果。

#### 18.2.3 背景色设置

```js
useEffect(() => {
  if (!scene) return
  scene.background = new THREE.Color('#FFFFFF')
}, [scene])
```

产品含义：

- 预览框使用白色背景。
- 便于黑色线框、图标和文字显示。

#### 18.2.4 小画布数据注册

```js
useEffect(() => {
  setSmallCanvasData(camera, scene, gl)
}, [scene, camera, gl])
```

产品含义：

- 将左上角小画布的相机、场景、renderer 保存到父组件。
- 保存分镜时，`use-save-to-storyboarder.js` 会使用这份数据渲染 `plot` 图。

#### 18.2.5 自动适配视野

核心函数：`autofitOrtho`

职责：根据场景中关键对象的位置自动调整正交相机范围。

关键逻辑：

```js
let minMax = [9999, -9999, 9999, -9999]

for (let i = 0, length = scene.children[0].children.length; i < length; i++) {
  let child = scene.children[0].children[i]
  if (
    child.userData &&
    child.userData.type === 'object' ||
    child.userData.type === 'character' ||
    child.userData.type === 'light' ||
    child.userData.type === 'volume' ||
    child.userData.type === 'image' ||
    child.userData.type === 'camera'
  ) {
    minMax[0] = Math.min(child.position.x, minMax[0])
    minMax[1] = Math.max(child.position.x, minMax[1])
    minMax[2] = Math.min(child.position.z, minMax[2])
    minMax[3] = Math.max(child.position.z, minMax[3])
    numVisible++
  }
}
```

之后设置正交相机范围：

```js
camera.position.x = minMax[0] + ((minMax[1] - minMax[0]) / 2)
camera.position.z = minMax[2] + ((minMax[3] - minMax[2]) / 2)
camera.left = -(minMax[1] - minMax[0]) / 2
camera.right = (minMax[1] - minMax[0]) / 2
camera.top = (minMax[3] - minMax[2]) / 2
camera.bottom = -(minMax[3] - minMax[2]) / 2
camera.near = -1000
camera.far = 1000
camera.updateProjectionMatrix()
```

触发时机：

```js
useEffect(autofitOrtho, [sceneObjects, aspectRatio, fontMesh, renderData])
```

产品含义：

- 场景对象变化时自动重新取景。
- 新增、删除、移动对象后，小地图尽量保持全局可见。
- 小框与主框互换后，也会根据当前比例重新适配。

#### 18.2.6 点击命中与选择

核心函数：`intersectLogic`

```js
raycaster.current.setFromCamera({ x, y }, camera)
var intersects = raycaster.current.intersectObjects(getIntersectable(), true)
```

命中后调用：

```js
onPointerDown({
  clientX: e.clientX,
  clientY: e.clientY,
  object: target.object
})
```

`onPointerDown` 中处理业务选择：

```js
selectObject(match.userData.id)
if (match.userData.type === "camera") {
  setActiveCamera(match.userData.id)
}
```

产品含义：

- 点击普通对象：选中对象。
- 点击相机：选中并激活相机。
- 点击空白或地面：取消选择。

#### 18.2.7 拖拽对象

```js
const { prepareDrag, drag, updateStore, endDrag } = useDraggingManager(true)
```

拖拽流程：

```js
prepareDrag(draggedObject.current, { x, y, camera, scene, selections: [match.userData.id] })
```

```js
drag({ x, y }, draggedObject.current, camera, selections)
updateStore(updateObjects)
```

```js
endDrag(updateObjects)
draggedObject.current = null
```

产品含义：

- 用户可在俯视图直接拖动物体、人物、灯光、相机等对象。
- 拖拽结果通过 `updateObjects` 写入 Redux。
- 主视图和属性面板实时同步。

#### 18.2.8 小画布渲染

```js
const renderer = useShadingEffect(
  gl,
  mainViewCamera === 'live' ? ShadingType.Outline : world.shadingMode,
  world.backgroundColor
)
```

```js
useFrame(({ scene, camera }) => {
  if (renderData) {
    renderer.current.render(renderData.scene, renderData.camera)
  } else {
    renderer.current.render(scene, camera)
  }
}, 1)
```

产品含义：

- 正常情况下渲染俯视图。
- 与主视图互换后，小框可渲染原主相机画面。

#### 18.2.9 俯视图最终渲染对象

```jsx
<ModelObject ... isIcon={true} />
<IconsComponent ... />
<CameraIcon ... />
<Room ... isTopDown={true} />
<RemoteClients Component={XRClient} />
```

产品含义：

- 模型物体、人物/灯光/图片/体积、相机、房间和远程客户端都在同一俯视图中展示。

### 18.3 相机图标与视锥

#### `src/js/shot-generator/components/Three/Icons/CameraIcon.js`

职责：负责俯视图中相机 icon、相机名称、焦距、高度和视锥线的绘制与更新。

关键渲染内容：

- 相机图标 sprite。
- 相机名称文本。
- 焦距与高度文本。
- 左右视锥线。
- 选中高亮状态。

#### 18.3.1 创建相机图标和文本

```js
iconsSprites.current = new IconSprites(type, fontMesh)
iconsSprites.current.addText(text, 1)
```

焦距和高度文本：

```js
fakeCamera.current.fov = sceneObject.fov
let valueText = fakeCamera.current.getFocalLength().toFixed(2) + "mm, " + sceneObject.z.toFixed(2) + "m"
iconsSprites.current.addText(valueText, 2, { x: 0.7, y: 0, z: 0.39 }, 0.0055)
```

产品含义：

- 相机标签由两行构成。
- 第一行为相机名称。
- 第二行为镜头焦距和相机高度。

#### 18.3.2 创建视锥线

```js
let hFOV = 2 * Math.atan(Math.tan(sceneObject.fov * Math.PI / 180 / 2) * 1)
frustumIcons.current.left.icon.material.rotation = hFOV / 2 + ref.current.rotation.y
frustumIcons.current.right.icon.material.rotation = -hFOV / 2 + ref.current.rotation.y
```

产品含义：

- 通过 FOV 计算相机水平视角。
- 用左右两条线表现相机视野范围。
- 相机旋转时，视锥线同步旋转。

#### 18.3.3 同步主相机实时状态

```js
useFrame(() => {
  if (!mainCamera || sceneObject.id !== mainCamera.userData.id || !iconsSprites.current || !fontMesh || mainCamera.isSynchronized) return false

  const currentRotation = new THREE.Euler()
    .setFromQuaternion(mainCamera.quaternion, 'YXZ').y

  iconsSprites.current.icon.material.rotation = currentRotation

  let hFOV = 2 * Math.atan(Math.tan(mainCamera.fov * Math.PI / 180 / 2) * 1)
  frustumIcons.current.left.icon.material.rotation = hFOV / 2 + currentRotation
  frustumIcons.current.right.icon.material.rotation = -hFOV / 2 + currentRotation

  let isPositionChanged = !ref.current.position.equals(mainCamera.position)
  if (isPositionChanged) {
    ref.current.position.copy(mainCamera.position)
    props.autofitOrtho()
  }

  mainCamera.isSynchronized = true
})
```

产品含义：

- 当用户通过主视图相机控制器移动当前相机时，左上角相机 icon 实时同步。
- 相机位置变化时触发 `autofitOrtho`，保证新位置仍在俯视图中可见。

#### 18.3.4 响应 sceneObject 更新

```js
useEffect(() => {
  if (fontMesh && ref.current) {
    iconsSprites.current.icon.material.rotation = sceneObject.rotation

    let hFOV = 2 * Math.atan(Math.tan(sceneObject.fov * Math.PI / 180 / 2) * 1)
    frustumIcons.current.left.icon.material.rotation = hFOV / 2 + sceneObject.rotation
    frustumIcons.current.right.icon.material.rotation = -hFOV / 2 + sceneObject.rotation

    fakeCamera.current.fov = sceneObject.fov
    let focal = fakeCamera.current.getFocalLength().toFixed(2)
    let meters = parseFloat(Math.round(sceneObject.z * 100) / 100).toFixed(2)
    if (iconsSprites.current.iconSecondText) {
      iconsSprites.current.changeSecondText(`${Math.round(focal)}mm, ${meters}m`)
    }
  }
}, [sceneObject])
```

产品含义：

- Redux 中相机对象变化后，图标朝向、视锥和文本应同步更新。
- 当前代码中存在焦距格式不完全统一的问题，可作为后续优化项。

#### 18.3.5 选中高亮

```js
useEffect(() => {
  if (!iconsSprites.current) return
  iconsSprites.current.setSelected(props.isSelected)
}, [props.isSelected])
```

产品含义：

- 当相机被选中时，相机 icon 变为高亮色。

### 18.4 普通对象图标

#### `src/js/shot-generator/components/IconsComponent/index.js`

职责：负责人物、灯光、体积、图片等非相机对象在俯视图中的图标和文字展示。

关键逻辑：

```js
iconsSprites.current = new IconsSprites(type, fontMesh)
text && iconsSprites.current.addText(text, 1, { x: 0.7, y: 0, z: 0 })
auxiliaryText && iconsSprites.current.addText(auxiliaryText, 2, { x: 0.7, y: 0, z: 0.39 })
iconsSprites.current.icon.material.rotation = translateRotation(sceneObject)
ref.current.add(iconsSprites.current)
```

对象位置映射：

```jsx
<group
  ref={ref}
  position={[x, z + 1, y]}
  visible={sceneObject.visible}
  userData={{
    type: type,
    id: sceneObject.id,
    name: sceneObject.name,
    locked: sceneObject.locked
  }}
>
```

产品含义：

- 普通对象在俯视图上显示为图标 + 名称。
- `userData` 用于点击命中后反查业务对象。
- `visible` 控制是否出现在俯视图。

旋转转换：

```js
const translateRotation = (sceneObject) => {
  switch (sceneObject.type) {
    case 'character':
      return sceneObject.rotation + Math.PI
    case 'light':
      let addRotation = sceneObject.tilt >= 0 ? 0 : Math.PI
      return sceneObject.rotation + addRotation
    case 'volume':
      return sceneObject.rotation
    case 'image':
      return sceneObject.rotation.y
  }
}
```

产品含义：

- 不同对象类型的业务旋转字段不完全一致，需要转换为俯视图 icon 的旋转角度。

### 18.5 图标 Sprite 与文本系统

#### `src/js/shot-generator/components/IconsComponent/IconSprites.js`

职责：封装 icon sprite、BMFont 文本和选中态颜色。

关键逻辑：

```js
class IconSprites extends Object3D {
  constructor(type, mesh) {
    super()
    let icon
    switch (type) {
      case 'character': icon = allSprites.character; break
      case 'camera': icon = allSprites.camera; break
      case 'light': icon = allSprites.light; break
      case 'object': icon = allSprites.object; break
      case 'volume': icon = allSprites.volume; break
      case 'image': icon = allSprites.image; break
    }
    this.mesh = mesh.clone()
    this.textMeshes = []
    this.icon = icon.clone()
    this.add(this.icon)
  }
}
```

添加文字：

```js
addText(text, index, position = { x: 0.7, y: 0, z: 0 }, scale = 0.006, rotation = { x: -Math.PI/2, y: Math.PI, z: Math.PI }) {
  let mesh = this.mesh.clone()
  mesh.geometry = createGeometry(mesh.geometry._opt)
  mesh.scale.set(scale, scale, scale)
  mesh.rotation.set(rotation.x, rotation.y, rotation.z)
  mesh.position.set(position.x, position.y, position.z)
  mesh.userData.text = text
  mesh.geometry.update(text)
  this.textMeshes.push({ key: index, value: mesh })
  this.add(mesh)
}
```

选中态：

```js
setSelected(value) {
  this.icon.material.color.set(value ? 0xff00ff : 0xffffff)
}
```

产品含义：

- 该文件定义俯视图图标的统一创建方式。
- 选中态目前使用洋红色 `0xff00ff`。
- 文本使用 BMFont 渲染，适合 Three.js 场景内显示。

#### `src/js/shot-generator/components/IconsComponent/IconContainer.js`

职责：提供不同类型对象对应的 sprite 资源。

产品含义：

- 相机、人物、灯光、物体、体积、图片等 icon 的底层资源从这里取得。

### 18.6 字体加载

#### `src/js/shot-generator/hooks/use-font-loader.js`

职责：加载俯视图中 icon 标签文本所需的 BMFont 字体资源。

在 `SceneManagerR3fSmall.js` 中使用：

```js
const fontpath = path.join(window.__dirname, '..', 'src', 'fonts', 'thicccboi-bmfont', 'thicccboi-bold.fnt')
const fontpnguri = 'fonts/thicccboi-bmfont/thicccboi-bold.png'
const fontMesh = useFontLoader(fontpath, fontpnguri)
```

产品含义：

- 字体加载成功后，俯视图中对象名称和相机参数才会显示。
- 字体未加载前，应至少保证 icon 不阻塞页面渲染。

### 18.7 拖拽管理

#### `src/js/shot-generator/hooks/use-dragging-manager.js`

职责：封装拖拽开始、拖拽计算、状态更新和拖拽结束逻辑。

在 `SceneManagerR3fSmall.js` 中使用：

```js
const { prepareDrag, drag, updateStore, endDrag } = useDraggingManager(true)
```

产品含义：

- `true` 表示当前拖拽逻辑服务于俯视图/小画布模式。
- 该 hook 决定对象在俯视图中被拖拽后如何映射到场景坐标。
- 拖拽完成后通过 `updateObjects` 写回 Redux。

### 18.8 主相机画面同步

#### `src/js/shot-generator/CameraUpdate.js`

职责：将 Redux 中 activeCamera 的状态同步到 Three.js 主相机。

关键逻辑：

```js
activeCamera: getSceneObjects(state)[getActiveCamera(state)]
```

产品含义：

- 俯视图点击相机后会调用 `setActiveCamera`。
- activeCamera 变化后，主视图相机同步到对应相机对象状态。
- 因此，点击左上角预览框中的相机可切换主视图取景。

#### `src/js/shot-generator/components/Three/CameraControlsComponet.js`

职责：主视图中相机控制器的状态同步和交互控制。

产品含义：

- 用户在主视图移动/旋转当前相机时，相机对象状态更新。
- `CameraIcon.js` 会通过 `mainCamera` 同步当前相机在俯视图中的位置、旋转和视锥。

### 18.9 左侧对象列表联动

#### `src/js/shot-generator/components/ItemList/index.js`

职责：左侧对象列表的数据排序、选择、隐藏、锁定、删除和相机激活。

相机选择逻辑：

```js
selectObject([...new Set(currentSelections)])
if (props.type === "camera") {
  setActiveCamera(props.id)
}
```

产品含义：

- 左侧列表点击相机和俯视图点击相机行为一致。
- 都会选中对象并切换当前激活相机。

隐藏逻辑：

```js
let nextVisibility = !props.visible
updateObject(props.id, { visible: nextVisibility })
```

产品含义：

- 对象在列表中被隐藏后，俯视图中的对应 icon 也会隐藏。

锁定逻辑：

```js
let nextAvailability = !props.locked
updateObject(props.id, { locked: nextAvailability })
```

产品含义：

- 对象锁定后，俯视图中不能再拖拽或选择该对象。

#### `src/js/shot-generator/components/ItemList/Item.js`

职责：单个对象列表项的展示。

产品含义：

- 显示对象类型 icon、名称、active 相机标识、锁定、隐藏和删除入口。
- 与俯视图共享 `selections`、`activeCamera`、`visible`、`locked` 状态。

### 18.10 Redux 状态与动作

#### `src/js/shared/reducers/shot-generator.js`

职责：Shot Generator 的核心 Redux reducer、selector 和 action creators。

与左上角预览框直接相关的 selector：

```js
const getSceneObjects = state => state.undoable.present.sceneObjects
const getSelections = state => state.undoable.present.selections
const getActiveCamera = state => state.undoable.present.activeCamera
const getWorld = state => state.undoable.present.world
```

与左上角预览框直接相关的 action creator：

```js
selectObject: id => ({ type: 'SELECT_OBJECT', payload: id })
updateObject: (id, values) => ({ type: 'UPDATE_OBJECT', payload: { id, ...values } })
updateObjects: payload => ({ type: 'UPDATE_OBJECTS', payload })
setActiveCamera: id => ({ type: 'SET_ACTIVE_CAMERA', payload: id })
```

activeCamera reducer：

```js
const activeCameraReducer = (state = initialScene.activeCamera, action) => {
  return produce(state, draft => {
    switch (action.type) {
      case 'LOAD_SCENE':
      case 'UPDATE_SCENE_FROM_XR':
        return action.payload.activeCamera

      case 'SET_ACTIVE_CAMERA':
        return action.payload

      default:
        return draft
    }
  })
}
```

产品含义：

- `sceneObjects` 决定俯视图中显示哪些对象。
- `selections` 决定俯视图中哪些对象高亮。
- `activeCamera` 决定主视图取景相机。
- `world` 决定房间、灯光、背景、shading 等环境表现。

### 18.11 创建相机与对象

#### `src/js/shared/actions/scene-object-creators.js`

职责：创建相机、人物、物体、灯光、体积和图片对象。

相机创建逻辑：

```js
const createCamera = (id, cameraState, camera) => {
  let { x, y, z } = camera.position
  let rotation = camera.rotation.y
  let tilt = camera.rotation.x
  let roll = camera.rotation.z

  x -= 0.91

  let object = {
    id,
    type: 'camera',
    fov: cameraState.fov,
    x,
    y: z,
    z: y,
    rotation,
    tilt,
    roll
  }

  return createObject(object)
}
```

产品含义：

- 新建相机会基于当前相机状态生成。
- 新相机在俯视图中会立即出现。
- 新相机会继承当前相机 FOV。
- 新相机位置会相对当前相机略微偏移，避免完全重叠。

#### `src/js/shot-generator/components/Toolbar/index.js`

职责：顶部工具栏中新增相机/对象/人物/灯光/图片等操作入口。

新增相机逻辑：

```js
const onCreateCameraClick = () => {
  let id = THREE.Math.generateUUID()

  initCamera()
  undoGroupStart()
  createCamera(id, cameraState, camera.current)
  selectObject(id)
  setActiveCamera(id)
  undoGroupEnd()
}
```

产品含义：

- 用户点击顶部“相机”按钮创建新相机。
- 新相机被选中并立即成为 activeCamera。
- 左上角预览框同步显示新相机。

### 18.12 保存到故事板与 plot 图生成

#### `src/js/shot-generator/hooks/use-save-to-storyboarder.js`

职责：保存当前镜头和插入新镜头时，渲染并导出主相机画面和俯视 plot 图。

核心函数：`renderAll`

```js
const renderAll = (
  shotRenderer,
  cameraPlotRenderer,
  largeCanvasData,
  smallCanvasData,
  shotSize,
  cameraPlotSize,
  aspectRatio
) => {
  shotRenderer.current.setSize(shotSize.width, shotSize.height)
  renderShot(
    shotRenderer.current,
    largeCanvasData.current.scene,
    largeCanvasData.current.camera,
    { size: shotSize, aspectRatio }
  )
  let shotImageDataUrl = shotRenderer.current.domElement.toDataURL()

  cameraPlotRenderer.current.setSize(cameraPlotSize.width, cameraPlotSize.height)
  renderCameraPlot(
    cameraPlotRenderer.current,
    smallCanvasData.current.scene,
    smallCanvasData.current.camera,
    { size: cameraPlotSize }
  )
  let cameraPlotImageDataUrl = cameraPlotRenderer.current.domElement.toDataURL()

  return {
    shotImageDataUrl,
    cameraPlotImageDataUrl
  }
}
```

plot 渲染：

```js
const renderCameraPlot = (renderer, scene, originalCamera) => {
  let camera = originalCamera.clone()

  camera.left = camera.bottom
  camera.right = camera.top

  setCameraAspectFromRendererSize(renderer, camera)
  camera.updateProjectionMatrix()

  renderer.render(scene, camera)
}
```

保存时发送数据：

```js
ipcRenderer.send('saveShot', {
  uid,
  data,
  images: {
    camera: shotImageDataUrl,
    plot: cameraPlotImageDataUrl
  }
})
```

产品含义：

- 左上角预览框不仅是交互 UI，也是最终 storyboard 数据中的 plot 图来源。
- 保存时会同时生成主相机图和俯视图。
- `smallCanvasData.current.scene` 和 `smallCanvasData.current.camera` 是 plot 图渲染的关键输入。

### 18.13 样式文件

#### `src/css/shot-generator.css`

职责：定义左上角预览框布局、尺寸、控件覆盖层和画布基础样式。

关键样式：

```css
#aside {
  display: flex;
  flex-direction: column;
  width: 300px;
  height: 100%;
  background: #111;
  flex-shrink: 0;
}

#topdown {
  display: flex;
  align-items: center;
  position: relative;
  height: 300px;
}

#top-down-canvas {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.topdown__controls {
  pointer-events: none;
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
}

.topdown__controls .row {
  pointer-events: auto;
}
```

产品含义：

- 左侧栏固定 300px 宽。
- 俯视预览框固定 300px 高。
- 展开按钮通过 absolute overlay 覆盖在画布右下角。
- overlay 默认不拦截事件，只有按钮行可点击。

### 18.14 房间与空间参考

#### `src/js/shot-generator/components/Three/Room.js`

职责：在俯视图中展示房间边界或空间参考。

调用位置：

```jsx
<Room
  width={world.room.width}
  length={world.room.length}
  height={world.room.height}
  visible={world.room.visible}
  isTopDown={true}
/>
```

产品含义：

- 当用户启用房间时，俯视图中可见空间边界。
- `isTopDown` 表示以俯视图模式渲染房间。

### 18.15 远程与 XR 客户端展示

#### `src/js/shot-generator/components/RemoteClients.js`

职责：管理远程客户端列表并在场景中渲染。

#### `src/js/shot-generator/components/Three/XRClient.js`

职责：在 Three.js 场景中展示 XR 客户端位置。

调用位置：

```jsx
<RemoteProvider>
  <RemoteClients Component={XRClient} />
</RemoteProvider>
```

产品含义：

- 当有远程或 VR/XR 客户端连接时，俯视图可显示客户端位置。
- 有助于协作场景中理解用户所在空间位置。

## 19. 代码调用链总结

### 19.1 页面初始化调用链

```text
Editor/index.js
  ↓
#topdown Canvas
  ↓
SceneManagerR3fSmall
  ↓
读取 Redux sceneObjects / world / selections
  ↓
初始化正交俯视相机
  ↓
渲染 ModelObject / IconsComponent / CameraIcon / Room / XRClient
```

### 19.2 点击相机调用链

```text
用户点击左上角相机 icon
  ↓
SceneManagerR3fSmall.intersectLogic
  ↓
Raycaster 命中 Three.js 对象
  ↓
onPointerDown
  ↓
selectObject(cameraId)
  ↓
setActiveCamera(cameraId)
  ↓
shot-generator reducer 更新 selections / activeCamera
  ↓
CameraUpdate / CameraControlsComponet 同步主视图相机
  ↓
左侧 ItemList 与俯视图高亮同步
```

### 19.3 拖拽对象调用链

```text
用户在俯视图 pointerdown
  ↓
SceneManagerR3fSmall.onPointerDown
  ↓
useDraggingManager.prepareDrag
  ↓
pointermove
  ↓
useDraggingManager.drag
  ↓
updateStore(updateObjects)
  ↓
Redux sceneObjects 更新
  ↓
主视图和俯视图重新渲染
  ↓
pointerup
  ↓
useDraggingManager.endDrag
```

### 19.4 保存 plot 图调用链

```text
用户点击保存到绘板 / 添加为新版
  ↓
Editor/index.js 监听 requestSaveShot / requestInsertShot
  ↓
use-save-to-storyboarder.js
  ↓
renderAll
  ↓
renderShot 使用 largeCanvasData 渲染主相机图
  ↓
renderCameraPlot 使用 smallCanvasData 渲染俯视 plot 图
  ↓
ipcRenderer.send('saveShot' / insertNewShot)
  ↓
主进程保存 camera + plot 图
```

## 20. 代码维护注意事项

- 修改 `SceneManagerR3fSmall.js` 的对象筛选逻辑时，需要同步确认 `autofitOrtho`、点击命中和渲染列表是否都覆盖新对象类型。
- 修改相机对象字段时，需要同时检查 `CameraIcon.js`、`CameraUpdate.js`、`CameraControlsComponet.js` 和 `scene-object-creators.js`。
- 修改选中态颜色时，应优先调整 `IconSprites.js` 的 `setSelected`，并检查主视图选中态是否需要保持一致。
- 修改左上角尺寸时，需要同步检查 `src/css/shot-generator.css` 中 `#aside`、`#topdown`、`#top-down-canvas`，以及 `SceneManagerR3fSmall.js` 中 renderer size 设置。
- 修改保存 plot 图比例时，需要检查 `use-save-to-storyboarder.js` 中 `renderCameraPlot` 和 `cameraPlotSize` 来源。
- 修改锁定对象行为时，需要同时检查 `SceneManagerR3fSmall.js`、`ItemList/index.js` 和 reducer 中 `updateObject` 对 locked/blocked 的处理。
- 修改主视图/小视图互换逻辑时，需要检查 `Editor/index.js` 中 `mainViewCamera`、`renderData`、`mainRenderData` 的传递关系。
