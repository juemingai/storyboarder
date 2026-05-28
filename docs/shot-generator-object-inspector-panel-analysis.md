# Shot Generator 添加物体后左侧属性面板功能与实现分析

## 1. 范围说明

本文档分析 Shot Generator 页面中点击工具栏 **Object** 添加物体，并在画布或左侧列表中选中该物体后，左侧属性面板展示的功能、交互流程和主要代码实现位置。

这里的“物体”特指 `sceneObject.type === 'object'` 的场景对象，不包含 Character、Camera、Light、Image、Volume 等其它类型。新增物体默认模型为 `box`。

## 2. 左侧面板整体结构

Shot Generator 主界面由 `Editor` 组件组织。左侧栏位于 `#aside`，由上到下包含：

1. 顶部俯视图 `#topdown`
2. 元素区域 `#elements`
3. 元素列表 `#listing`
4. 属性检查器 `#inspector`

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/Editor/index.js` | Shot Generator 主布局，渲染左侧 `#aside`、元素面板和主画布 |
| `src/js/shot-generator/components/ElementsPanel/index.js` | 左侧元素面板容器，连接 Redux，向列表和 Inspector 传递当前选择 |
| `src/js/shot-generator/components/ElementsPanel/Inspector.js` | 根据选择数量决定展示世界属性、单对象属性或多选属性 |
| `src/css/shot-generator.css` | 左侧栏、元素列表、属性面板、标签页、缩略图列表等样式 |

选择逻辑：

- 没有选中对象时：`Inspector` 渲染 `InspectedWorld`，展示场景/世界属性。
- 选中 1 个对象时：渲染 `InspectedElement`，展示该对象属性。
- 选中多个对象或组时：渲染 `MultiSelectionInspector`。

## 3. 添加物体到选中物体的流程

### 3.1 点击工具栏 Object

工具栏中 Object 按钮由 `Toolbar` 组件提供。

核心流程：

1. 点击 Object 按钮触发 `onCreateObjectClick`。
2. 生成 UUID。
3. 读取当前激活相机位置和朝向。
4. 调用 `createModelObject(id, camera, room)` 创建 `object` 类型场景对象。
5. 调用 `selectObject(id)` 立即选中新物体。
6. 通过 `undoGroupStart()` / `undoGroupEnd()` 将创建和选择纳入一组撤销操作。

相关代码：

| 文件 | 关键点 |
| --- | --- |
| `src/js/shot-generator/components/Toolbar/index.js` | `onCreateObjectClick`、Object 按钮 UI、创建后选中对象 |
| `src/js/shared/actions/scene-object-creators.js` | `createModelObject` 定义新物体默认数据 |
| `src/js/shared/reducers/shot-generator.js` | `createObject` action、`CREATE_OBJECT` reducer、`selectObject` action、`SELECT_OBJECT` reducer |

新增物体默认属性：

```js
{
  type: 'object',
  model: 'box',
  tintColor: '#000000',
  width: 1,
  height: 1,
  depth: 1,
  x,
  y,
  z,
  rotation: { x: 0, y: rotation, z: 0 },
  visible: true
}
```

### 3.2 点击画布或列表编辑物体

用户可以通过两条路径选中物体：

| 入口 | 代码 | 行为 |
| --- | --- | --- |
| 主画布点击 3D 物体 | `src/js/shot-generator/components/Three/InteractionManager.js` | 通过 raycast / GPU picker 命中对象，调用 `selectObject(target.userData.id)` |
| 左侧对象列表点击条目 | `src/js/shot-generator/components/ItemList/index.js` | 点击列表项后调用 `selectObject([id])`；Shift 支持多选 |

选中状态存储在 Redux 的 `undoable.present.selections` 中。`InspectedElement` 通过 `getSelections(state)[0]` 找到当前对象，并按对象类型决定展示哪些 Tab。

## 4. 物体属性面板包含的功能

物体被选中后，`InspectedElement` 会显示标题和标签页。对于 `type === 'object'`，实际包含两个可用 Tab：

1. 参数 Tab：通用参数编辑，由 `GeneralInspector` + `ObjectInspector` 实现。
2. 模型 Tab：模型选择/搜索/导入，由 `ModelInspector` 实现。

Character 专属的手势、姿势、附件、表情、头发 Tab 不会在普通物体上显示。

### 4.1 标题与重命名

面板顶部显示：

```text
{selectedName} Properties
```

点击标题会打开一个 Modal，用于输入新名称。确认后调用：

```js
updateObject(id, { displayName: changedName, name: changedName })
```

功能点：

- 显示当前物体名称，优先使用 `object.name`，否则使用 `object.displayName`。
- 点击标题弹出命名对话框。
- 输入并确认后更新 `displayName` 和 `name`。
- 标题宽度固定为 288px，超长名称通过 `textOverflow: ellipsis` 截断。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/index.js` | 标题、重命名 Modal、Tab 组合逻辑 |
| `src/js/shot-generator/components/Modal/index.js` | 通用弹窗组件 |
| `src/js/shared/reducers/shot-generator.js` | `UPDATE_OBJECT` 合并 `name` / `displayName` 到场景对象 |
| `src/css/shot-generator.css` | `.object-property-heading` 标题样式 |

### 4.2 参数 Tab：位置编辑

参数 Tab 中前三项为位置坐标：

| 控件 | 对应字段 | 范围 | 说明 |
| --- | --- | --- | --- |
| X | `sceneObject.x` | -30 到 30 | 世界坐标 X |
| Y | `sceneObject.y` | -30 到 30 | 世界坐标 Y；Three.js 渲染时映射到 Z 平面坐标 |
| Z | `sceneObject.z` | -30 到 30 | 高度；Three.js 渲染时映射到 Y 坐标 |

每个控件使用 `NumberSlider`。交互能力包括：

- 左右箭头微调。
- 横向拖动数值调整。
- 按住 Alt 拖动可更细粒度调整。
- 双击数值切换为文本输入。
- 文本支持英尺/英寸格式转换为米，例如通过 `imperialToMetric` 转换。
- 拖动开始/结束会触发 `undoGroupStart` / `undoGroupEnd`，支持整体撤销。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | 定义 X/Y/Z 控件和 `updateObject(id, { x/y/z })` |
| `src/js/shot-generator/components/NumberSlider/index.js` | 数值滑杆、拖动、双击输入、格式化、撤销分组 |
| `src/js/shot-generator/components/Three/ModelObject.js` | 渲染时将 `position={[x, z, y]}` 应用到 Three.js 对象 |

### 4.3 参数 Tab：尺寸编辑

尺寸控件根据当前模型不同分两种情况。

#### 默认 Box 模型

当 `sceneObject.model === 'box'` 时显示三个独立尺寸：

| 控件 | 对应字段 | 范围 | 说明 |
| --- | --- | --- | --- |
| Width | `width` | 0.025 到 5 | 宽度 |
| Height | `height` | 0.025 到 5 | 高度 |
| Depth | `depth` | 0.025 到 5 | 深度 |

每项调用独立 setter：

```js
updateObject(id, { width })
updateObject(id, { height })
updateObject(id, { depth })
```

#### 非 Box 模型

当 `sceneObject.model !== 'box'` 时，只显示统一 `Size` 控件。它以 `depth` 作为显示值，更新时同时写入：

```js
updateObject(id, { width: size, height: size, depth: size })
```

这样普通模型会按等比例方式缩放。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | Box 独立宽高深；非 Box 统一 Size |
| `src/js/shot-generator/components/Three/ModelObject.js` | 渲染时将 `scale={[width, height, depth]}` 应用到 Three.js 对象 |
| `src/js/shot-generator/components/NumberSlider/index.js` | `sizeConstraint` 保证输入尺寸不低于 `0.01` |

### 4.4 参数 Tab：旋转编辑

物体支持三轴旋转：

| 控件 | UI 标签 | 写入字段 | UI 单位 | 代码映射 |
| --- | --- | --- | --- | --- |
| Rotate X | `rotate-x` | `rotation.x` | 角度 | `degToRad(x)` |
| Rotate Y | `rotate-y` | `rotation.z` | 角度 | UI 的 Y 写入内部 `rotation.z` |
| Rotate Z | `rotate-z` | `rotation.y` | 角度 | UI 的 Z 写入内部 `rotation.y` |

需要注意：UI 上的 Y/Z 与内部字段存在轴映射关系，这是为了适配该项目中 Three.js 坐标与 Shot Generator 场景坐标的转换。

交互能力：

- 范围为 -180° 到 180°。
- 步进为 1°。
- 显示格式为 `N°`。
- 输入值通过 `degToRad` 转换为弧度存储。
- reducer 对 `object` 类型的 `rotation` 执行浅合并，避免只改一个轴时覆盖其它轴。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | Rotate X/Y/Z 控件和角度/弧度转换 |
| `src/js/shot-generator/components/NumberSlider/index.js` | `transforms.degrees`、`formatters.degrees` |
| `src/js/shared/reducers/shot-generator.js` | `updateObject` 中对 `object` / `image` 的 `rotation` 做 merge |
| `src/js/shot-generator/components/Three/ModelObject.js` | 渲染时 `rotation={[rotation.x, rotation.y, rotation.z]}` |

### 4.5 参数 Tab：颜色 Tint Color

物体支持设置 `tintColor`。

功能点：

- UI 使用原生 `<input type="color">`。
- 默认值为 `#000000`。
- 选择颜色后调用 `updateObject(id, { tintColor })`。
- 渲染时 `ModelObject` 遍历 mesh material，将 `sceneObject.tintColor` 写入材质的 `emissive`。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | 渲染 `ColorSelect` 并更新 `tintColor` |
| `src/js/shot-generator/components/ColorSelect/index.js` | 颜色选择控件 |
| `src/js/shot-generator/components/Three/ModelObject.js` | 将 `tintColor` 应用于模型材质 |

### 4.6 参数 Tab：Drop Object

参数 Tab 底部有 `Drop Object` 按钮。

功能点：

- 点击按钮发送 IPC：`ipcRenderer.send('shot-generator:object:drops')`。
- Electron 主进程接收后转发为 `shot-generator:object:drop`。
- `SceneManagerR3fLarge` 注册该 IPC 命令。
- 对当前选中的 object 调用 `dropObject(selection, droppingPlaces)`。
- 计算落地后的新位置后批量调用 `updateObjects(changes)` 写回 Redux。

Drop 的目标包括：

- 其它 `object`
- `character`
- `ground`

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | Drop Object 按钮 UI 和 IPC 发送 |
| `src/js/windows/shot-generator/main.js` | IPC 转发：`shot-generator:object:drops` -> `shot-generator:object:drop` |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | 注册 IPC 命令并执行 `onCommandDrop` |
| `src/js/utils/dropToObjects.js` | `dropObject` / `dropCharacter` 的落点计算逻辑 |
| `src/js/shared/reducers/shot-generator.js` | `updateObjects` 批量更新位置 |
| `src/css/shot-generator.css` | `.drop_button` 样式 |

### 4.7 模型 Tab：搜索和切换模型

物体的第二个 Tab 是模型选择 Tab。该 Tab 只在 `selectedType` 为 `object` 或 `character` 时显示；对普通物体来说，它只筛选 `type === 'object'` 的模型。

功能点：

- 从 Redux `state.models` 中取出所有模型。
- 按 `sceneObject.type` 过滤，仅显示 `object` 类型模型。
- 搜索框使用模型 `name` 和 `keywords` 建立搜索文本。
- 点击缩略图后调用 `updateObject(sceneObject.id, { model: model.id })`。
- 当前模型会在缩略图列表中显示选中状态。
- 模型缩略图路径由 `filepathFor(model).replace(/.glb$/, '.jpg')` 得到。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/index.js` | 对 object 显示 Model Tab |
| `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js` | 模型筛选、搜索、选择、自定义文件导入 |
| `src/js/shot-generator/components/InspectedElement/ModelInspectorItem/index.js` | 单个模型缩略图项 UI |
| `src/js/shot-generator/components/SearchList/index.js` | 搜索输入和过滤回调 |
| `src/js/shot-generator/components/Grid/index.js` | 缩略图网格列表 |
| `src/js/shot-generator/utils/filepathFor.js` | 模型资源路径解析 |
| `src/js/shot-generator/utils/InspectorElementsSettings.js` | 缩略图网格尺寸配置 |
| `src/css/shot-generator.css` | `.thumbnail-search`、`.thumbnail-search__item` 等样式 |

### 4.8 模型 Tab：导入自定义模型

模型 Tab 还提供文件选择能力。

功能点：

- 使用 `FileInput` 选择本地模型文件。
- 选择文件后通过 `CopyFile(storyboarderFilePath, filepath.file, sceneObject.type)` 将文件复制到项目/资源目录。
- 对普通物体直接更新 `model` 为复制后的路径。
- 自定义模型被识别后，按钮显示文件名的中间截断版本。
- 帮助按钮链接到自定义 3D 模型 Wiki。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js` | `onSelectFile`、`isCustom`、自定义文件显示逻辑 |
| `src/js/shot-generator/components/FileInput/index.js` | 文件选择控件 |
| `src/js/shot-generator/utils/CopyFile.js` | 复制自定义模型文件 |
| `src/js/services/model-loader.js` | `ModelLoader.isCustomModel` 判断自定义模型 |
| `src/js/shot-generator/helpers/isUserModel.js` | 判断用户模型；普通 object 切换时不需要重置骨骼 |
| `src/js/shot-generator/components/HelpButton/index.js` | 自定义模型帮助链接 |

## 5. 属性更新的数据流

物体属性面板的统一数据流如下：

```text
UI 控件操作
  -> updateObject(id, partialValues)
  -> Redux action: UPDATE_OBJECT
  -> sceneObjectsReducer 更新 undoable.present.sceneObjects[id]
  -> React/Redux connect 触发 Inspector 和 Three 对象重渲染
  -> ModelObject 将位置、缩放、旋转、颜色应用到 Three.js group / material
```

关键实现：

| 文件 | 作用 |
| --- | --- |
| `src/js/shared/reducers/shot-generator.js` | 定义 selector、action creator、`UPDATE_OBJECT` / `UPDATE_OBJECTS` reducer |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js` | 根据当前选中对象类型选择 `ObjectInspector` |
| `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js` | 将 UI 控件值转换为 `updateObject` payload |
| `src/js/shot-generator/components/Three/ModelObject.js` | 根据 sceneObject 渲染并更新 Three.js object |

Reducer 中与物体编辑相关的重要行为：

- 如果对象 `locked` 或 `blocked`，除 `locked` 本身以外的更新会被忽略。
- `rotation` 对 `object` 和 `image` 是 merge 更新。
- `model` 更新后会重新计算 `displayName`。
- 普通字段会按 key 直接写入 draft。

## 6. 主场景紫色圆环旋转控制器

截图中物体前方的紫色圆环是选中物体后挂载到 Three.js 场景里的旋转控制器。它不是左侧 Inspector 的 DOM UI，而是 `TransformControls` 在 WebGL 主画布中绘制的 3D gizmo。普通物体选中后，用户可以直接拖动圆环绕 X/Y/Z 本地轴旋转物体，松手后把旋转结果写回 Redux。

### 6.1 初始化与生命周期

`SceneManagerR3fLarge` 在主场景初始化时创建一个全局复用的 `ObjectRotationControl`：

```js
objectRotationControl.current = new ObjectRotationControl(scene.children[0], camera, gl.domElement)
objectRotationControl.current.control.canSwitch = false
objectRotationControl.current.isEnabled = true
```

生命周期要点：

- 控制器挂在 `scene.children[0]` 这个主场景 root 下。
- 内部使用当前 Three camera 和 canvas `gl.domElement` 绑定指针事件。
- `activeGL` 变化时会重新绑定 DOM element，保证主视图/切换视图后仍能交互。
- camera 变化时调用 `setCamera(camera)`，让 gizmo 的 raycast 和朝向计算使用最新相机。
- 组件卸载时调用 `cleanUp()` 清理控制器、事件和辅助对象。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | 创建、持有、传递、清理 `objectRotationControl` |
| `src/js/shared/IK/objects/ObjectRotationControl.js` | 封装旋转控制器的 attach/detach、事件、状态写回回调 |
| `src/js/shared/IK/utils/TransformControls.js` | 底层 3D gizmo、raycast、旋转算法和指针事件 |

### 6.2 选中物体后挂载到 ModelObject

普通物体的 3D 渲染组件是 `ModelObject`。当 `isSelected === true` 时，组件会把全局 `objectRotationControl` attach 到当前物体的 Three group 上：

```js
props.objectRotationControl.setUpdateCharacter((name, rotation) => {
  let euler = new THREE.Euler().setFromQuaternion(ref.current.worldQuaternion())
  props.updateObject(ref.current.userData.id, {
    rotation: { x: euler.x, y: euler.y, z: euler.z }
  })
})
props.objectRotationControl.setCharacterId(ref.current.uuid)
props.objectRotationControl.selectObject(ref.current, ref.current.uuid)
props.objectRotationControl.control.setShownAxis(axis.X_axis | axis.Y_axis | axis.Z_axis)
props.objectRotationControl.IsEnabled = !sceneObject.locked
```

这里的命名 `setUpdateCharacter` / `setCharacterId` 是历史命名，普通物体也复用这套接口。对普通物体来说，它实际做的是：

1. 将 TransformControls 绑定到当前物体 group。
2. 显示 X/Y/Z 三轴旋转圆环。
3. 按锁定状态启用或禁用控制器。
4. 用户松手后读取物体世界 quaternion，转成 Euler rotation。
5. 调用 `updateObject(id, { rotation })` 写回场景对象。

取消选中或组件卸载时，如果当前控制器绑定的是该物体，会调用 `deselectObject()` 解除绑定。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/Three/ModelObject.js` | 选中物体后 attach 控制器；松手回写 rotation；取消选中时 detach |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | 将 `objectRotationControl.current` 作为 prop 传给 `ModelObject` |
| `src/js/shared/reducers/shot-generator.js` | `updateObject` 对 `object.rotation` 做 merge 更新 |

### 6.3 ObjectRotationControl 封装逻辑

`ObjectRotationControl` 是对自定义 `TransformControls` 的轻量封装。构造时固定为旋转模式：

```js
this.control = new TransformControls(camera, domElement)
this.control.rotationOnly = true
this.control.setSpace("local")
this.control.setMode('rotate')
this.control.size = 0.2
this.control.userData.type = "objectControl"
```

关键行为：

| 方法/字段 | 说明 |
| --- | --- |
| `selectObject(object, hitmeshid, offset)` | 将控制器 attach 到目标 Three object，添加到场景并绑定事件 |
| `deselectObject()` | detach 控制器，移出场景，移除事件监听 |
| `setUpdateCharacter(fn)` | 设置松手后回写 rotation 的回调 |
| `setCamera(camera)` | 更新控制器使用的相机 |
| `IsEnabled` | 设置 `TransformControls.enabled`，锁定物体时禁用交互 |
| `onMouseDown` | 标记 `this.object.isRotated = true` |
| `onMouseUp` | 调用回写回调，并标记 `isRotated = false` |

`ObjectRotationControl` 还会创建一个 `offsetObject`，并设置 `userData.type = 'controlTarget'`。在普通物体场景中通常 offset 为 `null`，控制器直接位于物体本地原点；角色等对象可传 offset 将控制器移到更合适的位置。

### 6.4 紫色圆环的绘制与交互

紫色圆环由 `TransformControlsGizmo` 生成。当前项目禁用了 scale：

```js
const isScaleDisabled = true
```

旋转 gizmo 使用 `TorusBufferGeometry` 画圆环：

```js
new THREE.TorusBufferGeometry(rotationalGizmoRadius, rotationalGizmoTube, 4, tubularSegments)
```

三轴颜色都使用紫色系材质：

| 轴 | 材质颜色 | 截图表现 |
| --- | --- | --- |
| X | `0x640AA1` | 紫色圆环之一 |
| Y | `0x6D22A1` | 紫色圆环之一 |
| Z | `0x8951B0` | 紫色圆环之一 |

参数设计：

| 参数 | 值 | 作用 |
| --- | --- | --- |
| `rotationalGizmoRadius` | `1.3` | 圆环半径 |
| `rotationalGizmoTube` | `radius / 12 + pickerTolerance` | 圆环粗细，同时扩大可点选区域 |
| `pickerTolerance` | `0.05` | 让圆环更容易被鼠标命中 |
| `tubularSegments` | `50` | 圆环分段数 |
| `control.size` | `0.2` | 控制器整体尺寸系数 |

`setShownAxis(axis.X_axis | axis.Y_axis | axis.Z_axis)` 会确保普通物体显示三轴旋转。相比之下，角色和 group 在其它组件中通常只显示 Y 轴旋转。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shared/IK/utils/TransformControls.js` | `TransformControlsGizmo`、紫色材质、torus 几何、picker 几何、轴显示控制 |
| `src/js/shot-generator/components/Three/ModelObject.js` | 普通物体显示 X/Y/Z 三轴控制器 |
| `src/js/shot-generator/components/Three/Character.js` | 角色复用同一控制器，但通常只显示 Y 轴 |
| `src/js/shot-generator/components/Three/Group.js` | group 复用同一控制器，但通常只显示 Y 轴 |

### 6.5 鼠标拖动旋转的数据流

主画布中拖动圆环的完整链路：

```text
用户 pointerdown 命中紫色圆环
  -> TransformControls.pointerHover 通过 picker 判断 axis
  -> TransformControls.pointerDown 记录起始 quaternion / pointer / plane intersection
  -> dispatch transformMouseDown
  -> ObjectRotationControl.onMouseDown 标记 object.isRotated = true
  -> pointermove 时 TransformControls.pointerMove 计算 rotationAngle
  -> 直接修改 Three object.quaternion
  -> pointerup 时 TransformControls.pointerUp
  -> dispatch transformMouseUp
  -> ObjectRotationControl.onMouseUp
  -> ModelObject 注册的回调读取 ref.current.worldQuaternion()
  -> updateObject(id, { rotation: { x, y, z } })
  -> Redux UPDATE_OBJECT
  -> sceneObject.rotation 更新
  -> ModelObject 下次渲染使用 rotation={[rotation.x, rotation.y, rotation.z]}
```

旋转算法核心在 `TransformControls.pointerMove`：

- 通过 raycast 计算当前 pointer 与控制平面的交点。
- 根据起点和终点计算 `offset`。
- 当前轴为 X/Y/Z 时，根据相机距离计算 `ROTATION_SPEED`。
- 在 local space 下，用 `object.quaternion.multiply(axisAngleQuaternion)` 更新物体。
- 更新过程中 dispatch `change` 和 `objectChange` 事件。

### 6.6 与主场景点击/拖拽管理的关系

紫色圆环本身也参与主场景命中检测。`TransformControls` 及其子节点都被标记为：

```js
userData.type = 'objectControl'
```

`InteractionManager` 对 `objectControl` 有专门处理：

- pointerdown 命中 `objectControl` 时，不走普通物体选择逻辑。
- 如果控制器目标是普通 object，会设置 `dragTarget` 且 `isObjectControl: true` 后 return。
- `onPointerMove` 中只有 `!dragTarget.isObjectControl` 才执行普通平移拖拽，因此拖圆环时不会触发物体平移。
- pointerup 阶段会忽略 `gizmo` / `objectControl` 命中，避免松手时误切换选中对象。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/Three/InteractionManager.js` | 识别 `objectControl`，阻止旋转控制器被当成普通物体拖拽/选择 |
| `src/js/shared/IK/objects/ObjectRotationControl.js` | 将控制器节点标记为 `objectControl` |
| `src/js/shared/IK/utils/TransformControls.js` | 绑定 pointerdown/pointermove/pointerup，发出 transform 事件 |

### 6.7 与左侧属性面板旋转控件的关系

左侧属性面板里的 Rotate X/Y/Z 和主场景紫色圆环最终都写入同一个字段：

```js
sceneObject.rotation = { x, y, z }
```

差异：

| 入口 | 更新方式 | 写入时机 |
| --- | --- | --- |
| 左侧 Rotate X/Y/Z NumberSlider | UI 数值直接转换成弧度，调用 `updateObject` | 数值变化时立即写入 |
| 主场景紫色圆环 | 拖动时先直接修改 Three object quaternion，松手后转 Euler 并 `updateObject` | `transformMouseUp` 时写入 |

因此两者是同一套状态的两种编辑入口：面板适合精确输入，主场景圆环适合视觉化快速调整。


## 7. 属性面板 UI 布局设计分析

本节聚焦物体选中后的左侧属性面板 UI 布局，而不是业务数据流。该面板整体采用“左侧固定宽度工具面板 + 主画布大面积预览”的桌面端 3D 编辑器布局，视觉上优先保证主画布面积，同时把对象列表和属性编辑压缩在左侧 300px 宽的 inspector 区域内。

### 7.1 左侧栏宏观布局

左侧栏由 `Editor` 中的 `#aside` 承载：

```jsx
<div id="aside">
  <div id="topdown">...</div>
  <div id="elements">
    <ElementsPanel/>
  </div>
</div>
```

对应 CSS：

```css
#aside {
  display: flex;
  flex-direction: column;
  width: 300px;
  height: 100%;
  background: #111;
  flex-shrink: 0;
}
```

布局特点：

| 设计点 | 实现 | 目的 |
| --- | --- | --- |
| 固定宽度 | `width: 300px` | 保持主画布和属性区比例稳定，避免编辑器 UI 抖动 |
| 垂直分区 | `flex-direction: column` | 顶部俯视图、下方元素/属性面板自然堆叠 |
| 不参与收缩 | `flex-shrink: 0` | 窗口变化时左侧工具区不被压窄 |
| 深色底色 | `background: #111` / `#333639` | 与中间浅色 3D 画布形成清晰对比 |

从截图看，左侧栏承担两个角色：上方是辅助导航/俯视定位，下方是对象管理和属性编辑。因此 `#topdown` 与 `#elements` 是纵向信息层级的第一层分割。

相关文件：

- `src/js/shot-generator/components/Editor/index.js`
- `src/css/shot-generator.css`

### 7.2 ElementsPanel：对象列表与属性面板的上下分割

`ElementsPanel` 内部使用一个 column flex 容器：

```jsx
<div style={{ flex: 1, display: "flex", flexDirection: "column" }}>
  <div id="listing">
    <ItemList/>
  </div>
  <Inspector ... />
</div>
```

对应 CSS：

```css
#listing {
  height: 200px;
  box-sizing: border-box;
  max-width: 300px;
}

#inspector {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100%;
  padding: 6px;
  background-color: #333639;
  color: #999;
  text-transform: capitalize;
  min-height: 0;
}
```

布局含义：

| 区域 | 尺寸策略 | UI 目的 |
| --- | --- | --- |
| `#listing` | 固定高度 200px | 保持对象列表始终可见，方便快速切换对象 |
| `#inspector` | `flex: 1` 占用剩余空间 | 根据选中对象显示可滚动属性内容 |
| `min-height: 0` | 允许 flex 子项正确滚动 | 避免 Inspector 内容把左侧栏撑爆 |

这种设计适合 3D 编辑器：用户需要频繁在对象列表与属性编辑之间切换，固定列表高度可以减少认知成本；属性区域则根据内容多少滚动。

相关文件：

- `src/js/shot-generator/components/ElementsPanel/index.js`
- `src/js/shot-generator/components/ElementsPanel/Inspector.js`
- `src/css/shot-generator.css`

### 7.3 对象列表 ItemList 的视觉层级

对象列表使用 `ItemList` 和 `Item` 渲染。列表第一项固定是 `Scene`，后面按类型排序显示 camera、character、object、image、light、volume、group。

列表条目的核心样式：

```css
.element {
  box-sizing: border-box;
  display: flex;
  flex-direction: row;
  background-color: rgba(255,255,255,0.01);
  font-size: 14px;
  letter-spacing: 0.05rem;
  max-width: 300px;
  height: 40px;
  overflow: hidden;
}

.element.selected {
  background-color: #5c65c0;
}
```

UI 设计点：

| 设计点 | 表现 | 作用 |
| --- | --- | --- |
| 行高固定 | `height: 40px` | 列表密度高，方便容纳多个对象 |
| 选中态高亮 | 蓝紫色背景 `#5c65c0` | 明确当前 Inspector 绑定的对象 |
| hover 显示操作 | `.hide-unless-hovered` | 降低锁定、隐藏、删除按钮的常驻视觉噪音 |
| 类型图标 + 名称 | `Item` 内部组合 | 快速区分 Camera / Object / Character 等对象 |
| 隐藏滚动条 | `::-webkit-scrollbar { display: none; }` | 保持工具面板更干净 |

相关文件：

- `src/js/shot-generator/components/ItemList/index.js`
- `src/js/shot-generator/components/ItemList/Item.js`
- `src/css/shot-generator.css`

### 7.4 Inspector 单对象布局

选中一个普通物体时，`ElementsPanel/Inspector.js` 会渲染 `InspectedElement`。它的 UI 层级是：

```text
#inspector
  -> object-property-heading
  -> Tabs
    -> tabs-header
      -> tabs-tab 参数图标
      -> tabs-tab 模型图标
    -> tabs-body
      -> Panel GeneralInspector / ModelInspector
```

`InspectedElement` 的标题：

```jsx
<a
  href="#"
  className="object-property-heading"
  style={{ overflow: "hidden", textOverflow: "ellipsis", flexShrink: 0, width: 288 }}
>
  {selectedName} Properties
</a>
```

布局特点：

| 层级 | UI 设计 | 作用 |
| --- | --- | --- |
| 标题 | 18px、粗体、低对比白色 | 告诉用户当前编辑对象，同时可点击重命名 |
| 固定宽度 288px | 与 `#inspector` 300px 减去 padding 对齐 | 避免文字撑破面板 |
| ellipsis | 超长名称截断 | 保持标题单行，不挤压 Tab |
| Tab header | 横向图标标签 | 用图标压缩多类属性入口，适配窄面板 |
| Tab body | 只渲染当前激活 Panel | 降低 DOM 和视觉复杂度 |

普通物体只显示两个关键 Tab：

| Tab | 图标来源 | 内容 |
| --- | --- | --- |
| 参数 Tab | `icon-tab-parameters` | 位置、尺寸、旋转、颜色、Drop Object |
| 模型 Tab | `icon-tab-model` | 搜索模型、切换模型、导入自定义模型 |

相关文件：

- `src/js/shot-generator/components/InspectedElement/index.js`
- `src/js/shot-generator/components/Tabs/index.js`
- `src/css/shot-generator.css`

### 7.5 Tabs 布局设计

Tab 组件是轻量自定义实现，不依赖外部 UI 库。`Tabs` 用 context 记录当前激活 index，`Tab` 点击后切换 index，`Panel` 只在激活时渲染。

Tab header 样式：

```css
.tabs-header {
  display: flex;
  flex-shrink: 0;
  width: 100%;
  overflow-x: hidden;
  border-bottom: 2px solid rgba(255,255,255,0.05);
}

.tabs-tab {
  background-color: rgba(255,255,255,0.05);
  padding: 4px;
  border-top-left-radius: 8px;
  border-top-right-radius: 8px;
  margin-right: 3px;
  display: flex;
  justify-content: center;
  align-items: center;
}
```

UI 设计点：

| 设计点 | 实现 | 目的 |
| --- | --- | --- |
| 图标优先 | Tab 内只放 `Icon` | 300px 窄面板中节省横向空间 |
| 顶部圆角 | `border-top-left/right-radius: 8px` | 表达“标签页”语义 |
| 轻微分割线 | `border-bottom` 低透明白色 | 分隔 Tab header 和内容区 |
| 不滚动横向溢出 | `overflow-x: hidden` | 防止窄面板出现横向滚动条 |
| 仅渲染 active Panel | `Panel` inactive 返回 `null` | 避免隐藏面板继续占布局 |

相关文件：

- `src/js/shot-generator/components/Tabs/index.js`
- `src/css/shot-generator.css`

### 7.6 参数 Tab 的表单布局

参数 Tab 由 `GeneralInspector` 选择 `ObjectInspector` 渲染，并用 `Scrollable` 包裹：

```jsx
<Scrollable>
  <ObjectInspector ... />
</Scrollable>
```

普通物体的参数控件按功能分组纵向排列：

```text
X
Y
Z
Width / Height / Depth 或 Size
Rotate X
Rotate Y
Rotate Z
Tint Color
Drop Object
```

布局设计意图：

| 顺序 | 分组 | 设计原因 |
| --- | --- | --- |
| 1 | 位置 X/Y/Z | 最常用空间属性，放在最上方 |
| 2 | 尺寸 Width/Height/Depth 或 Size | 紧跟位置，符合 transform 属性编辑习惯 |
| 3 | 旋转 Rotate X/Y/Z | 作为第三组 transform 属性 |
| 4 | Tint Color | 外观属性，优先级低于空间属性 |
| 5 | Drop Object | 操作型按钮，放底部避免误触 |

这种顺序接近 3D 软件中的 Transform 面板：Position -> Scale -> Rotation -> Appearance -> Action。

相关文件：

- `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js`
- `src/js/shot-generator/components/Scrollable/index.js`

### 7.7 NumberSlider 控件内部布局

`NumberSlider` 是属性面板最核心的控件。它不是原生 range，而是“label + 左箭头 + 数值框 + 右箭头”的紧凑编辑器控件。

DOM 结构：

```text
.number-slider
  -> .number-slider__label
  -> .number-slider__control
    -> .number-slider__nudge--left
    -> .number-slider__input
    -> .number-slider__nudge--right
```

交互设计：

| 交互 | UI 行为 | 价值 |
| --- | --- | --- |
| 点击左右箭头 | 按 step 微调 | 精细调整，不需要键盘 |
| 横向拖动数值框 | pointer lock 后连续改变值 | 类似专业 3D 软件属性拖拽体验 |
| Alt + 拖动 | 更小步进 | 精调 |
| 双击数值 | 切换为文本输入 | 支持精确输入 |
| Enter | 提交输入 | 键盘确认 |
| Escape | 取消输入 | 防止误输入 |

该控件的设计适合窄面板：相比传统 label + input + slider 三件套，它节省高度和宽度，同时保留拖动与精确输入能力。

相关文件：

- `src/js/shot-generator/components/NumberSlider/index.js`
- `src/css/shot-generator.css`

### 7.8 ColorSelect 与 Drop Button 布局

`ColorSelect` 是简单的 label + color input 结构：

```text
.color-select
  -> .color-select__label
  -> .color-select__control
    -> input[type="color"]
```

它保持与 `NumberSlider` 类似的“左侧语义 label + 右侧可操作控件”的表单节奏。

`Drop Object` 是操作型按钮，使用满宽横向布局：

```css
.drop_button__wrapper {
  display: flex;
  flex-direction: row;
  padding-bottom: 6px;
}

.drop_button {
  display: flex;
  justify-content: center;
  align-items: center;
  flex: 1;
  width: 100%;
  height: 26px;
  border-radius: 6px;
}
```

UI 设计点：

| 控件 | 布局策略 | 目的 |
| --- | --- | --- |
| ColorSelect | 与表单控件保持一致节奏 | 外观属性不破坏参数面板扫描方式 |
| Drop Button | 满宽按钮 | 强调这是一次性动作而不是连续参数 |
| hover 加亮 | `rgba(255,255,255,0.2)` | 反馈可点击 |
| 低对比默认态 | `rgba(255,255,255,0.05)` | 避免和高频参数控件争夺注意力 |

相关文件：

- `src/js/shot-generator/components/ColorSelect/index.js`
- `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js`
- `src/css/shot-generator.css`

### 7.9 模型 Tab 的搜索与缩略图网格布局

模型 Tab 使用 `.thumbnail-search` 这套通用缩略图面板布局。该布局也被 Pose、Emotion、Hair 等面板复用。

结构：

```text
.thumbnail-search.column
  -> 顶部 row
    -> SearchList
    -> or 文案 / FileInput
    -> HelpButton
  -> .thumbnail-search__list
    -> Scrollable
      -> Grid
        -> ModelInspectorItem
```

关键 CSS：

```css
.thumbnail-search {
  width: 288px;
  height: 100%;
  margin-bottom: 10px;
  text-transform: none;
}

.thumbnail-search input {
  width: 100%;
  padding: 8px 6px;
  border: none;
  background-color: rgba(255,255,255,0.1);
  color: #999;
  border-radius: 6px;
}

.thumbnail-search__list {
  height: 100%;
  overflow: auto;
}
```

UI 设计点：

| 区域 | 设计 | 目的 |
| --- | --- | --- |
| 顶部搜索行 | 搜索框占满剩余空间 | 模型数量较多时优先搜索 |
| FileInput | 固定约 106px 宽 | 保持上传入口稳定可见 |
| HelpButton | 右侧小图标 | 不打断主流程，但保留说明入口 |
| 缩略图 | 图片 + 名称 | 比纯文字更适合选择 3D 模型 |
| 选中态 | 名称变白 | 在暗色背景下轻量表达状态 |
| hover 态 | 图片 opacity 提升、文字变亮 | 提供可点击反馈 |

相关文件：

- `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/ModelInspectorItem/index.js`
- `src/js/shot-generator/components/SearchList/index.js`
- `src/js/shot-generator/components/Grid/index.js`
- `src/js/shot-generator/components/FileInput/index.js`
- `src/css/shot-generator.css`

### 7.10 Grid 缩略图布局与性能设计

`Grid` 并不是一次性渲染所有模型项，而是根据滚动位置逐步渲染：

- `getIndices(container, itemHeight, numCols)` 计算当前需要渲染到的 index。
- 父容器滚动时 throttle 更新 `currentIndex`。
- 未到渲染范围的 item 使用 placeholder 占位，保持整体高度和网格结构稳定。

布局价值：

| 设计点 | 作用 |
| --- | --- |
| 固定 `itemHeight` | 保证虚拟/延迟渲染时可预测布局 |
| `numCols` | 让缩略图按固定列数排布 |
| placeholder | 保持滚动高度和网格行不跳动 |
| throttle scroll | 降低滚动时频繁重渲染 |

相关文件：

- `src/js/shot-generator/components/Grid/index.js`
- `src/js/shot-generator/utils/InspectorElementsSettings.js`

### 7.11 Modal 重命名/保存预设布局

物体标题点击重命名时使用通用 `Modal`。虽然不是物体参数面板常驻内容，但属于属性面板的延伸交互。

布局结构：

```text
Modal
  -> 描述文案
  -> input.modalInput
  -> .skeleton-selector__div
    -> button.skeleton-selector__button
```

设计特点：

| 设计点 | 说明 |
| --- | --- |
| 深色输入框 | 与 Inspector 暗色系统保持一致 |
| 居中按钮区 | 弹窗操作明确，避免和正文混在一起 |
| 蓝紫色主按钮 | 与选中态 `#5c65c0` 保持一致 |
| placeholder 使用当前名称 | 降低用户理解成本 |

相关文件：

- `src/js/shot-generator/components/InspectedElement/index.js`
- `src/js/shot-generator/components/Modal/index.js`
- `src/css/shot-generator.css`

### 7.12 状态视觉：selected、hover、locked、disabled

属性面板和相关主场景 UI 使用较少但明确的状态层级：

| 状态 | UI 表现 | 代码位置 |
| --- | --- | --- |
| 列表选中 | `.element.selected` 蓝紫色背景 | `ItemList` / `Item` / CSS |
| Tab 激活 | `.tabs-tab active` | `Tabs/index.js` |
| 模型选中 | `.thumbnail-search__item--selected` | `ModelInspectorItem` |
| hover 可操作 | 背景或 opacity 提升 | CSS hover 规则 |
| locked 物体 | 列表锁图标；旋转控制器 `IsEnabled = false` | `ItemList`、`ModelObject`、`ObjectRotationControl` |
| blocked 物体 | 3D outline helper 使用 blocked 状态 | `ModelObject`、`outlineMaterial` |

这套状态设计的特点是：高频状态（选中）用强色块，低频操作（hover、删除、帮助）默认弱化，避免左侧面板变成一组高噪音按钮。

### 7.13 复刻/迁移时建议的组件拆分

如果要在新项目中复刻这套物体属性面板，建议按下面方式拆分：

```text
ShotGeneratorLayout
  -> LeftSidebar
    -> TopDownPreview
    -> ElementsPanel
      -> ObjectList
      -> InspectorHost
        -> InspectorHeader
        -> InspectorTabs
          -> ObjectGeneralPanel
            -> TransformSection
            -> AppearanceSection
            -> ActionSection
          -> ObjectModelPanel
            -> ModelSearchBar
            -> CustomModelPicker
            -> ModelGrid
  -> MainViewport
    -> SceneCanvas
    -> ObjectRotationGizmo
```

迁移注意点：

| 模块 | 建议 |
| --- | --- |
| 布局 | 保留左侧固定宽度 + 主画布 flex fill 的编辑器结构 |
| Inspector | 保持标题、Tab、Panel 三层结构，避免单文件过大 |
| 表单控件 | `NumberSlider` 建议独立成通用控件，因为位置/尺寸/旋转都依赖它 |
| 缩略图网格 | 模型、姿势、表情都可复用同一套 Search + Grid 模式 |
| 3D gizmo | 不应放入 DOM Inspector，应作为 SceneCanvas 的 overlay/Three object 处理 |
| 状态 | 面板输入和主场景 gizmo 最终应写同一份 sceneObject transform state |

## 8. 相关 UI 样式文件

主要样式集中在 `src/css/shot-generator.css`：

| 样式选择器 | 用途 |
| --- | --- |
| `#aside` | 左侧栏尺寸和背景 |
| `#elements` | 左侧元素区域容器 |
| `#listing` | 元素列表区域 |
| `.objects-list` / `.element` | 对象列表条目样式、选中态、删除/锁定/隐藏按钮 |
| `#inspector` | 属性检查器容器 |
| `.object-property-heading` | Inspector 顶部标题 |
| `.tabs-header` / `.tabs-tab` / `.tabs-body__content` | Tab 头和内容区域 |
| `.number-slider*` | 数字滑杆控件样式 |
| `.color-select*` | 颜色选择控件样式 |
| `.drop_button` | Drop Object 按钮样式 |
| `.thumbnail-search*` | 模型搜索、缩略图网格、文件按钮样式 |

## 9. 文件清单

### 9.1 入口和布局

- `src/js/shot-generator/components/Editor/index.js`
- `src/js/shot-generator/components/Toolbar/index.js`
- `src/js/shot-generator/components/ElementsPanel/index.js`
- `src/js/shot-generator/components/ElementsPanel/Inspector.js`
- `src/css/shot-generator.css`

### 9.2 Inspector UI

- `src/js/shot-generator/components/InspectedElement/index.js`
- `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/GeneralInspector/Object.js`
- `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/ModelInspectorItem/index.js`
- `src/js/shot-generator/components/NumberSlider/index.js`
- `src/js/shot-generator/components/ColorSelect/index.js`
- `src/js/shot-generator/components/Tabs/index.js`
- `src/js/shot-generator/components/FileInput/index.js`
- `src/js/shot-generator/components/SearchList/index.js`
- `src/js/shot-generator/components/Grid/index.js`
- `src/js/shot-generator/components/Scrollable/index.js`
- `src/js/shot-generator/components/Modal/index.js`
- `src/js/shot-generator/components/HelpButton/index.js`

### 9.3 3D 交互和渲染

- `src/js/shot-generator/components/Three/InteractionManager.js`
- `src/js/shot-generator/components/Three/ModelObject.js`
- `src/js/shot-generator/SceneManagerR3fLarge.js`
- `src/js/shared/IK/objects/ObjectRotationControl.js`
- `src/js/shared/IK/utils/TransformControls.js`
- `src/js/shot-generator/hooks/use-dragging-manager.js`
- `src/js/utils/dropToObjects.js`

### 9.4 状态和资源

- `src/js/shared/actions/scene-object-creators.js`
- `src/js/shared/reducers/shot-generator.js`
- `src/js/services/model-loader.js`
- `src/js/shot-generator/utils/CopyFile.js`
- `src/js/shot-generator/utils/filepathFor.js`
- `src/js/shot-generator/utils/InspectorElementsSettings.js`
- `src/js/windows/shot-generator/main.js`

## 10. 功能总结

添加物体并进入编辑后，左侧属性面板提供以下能力：

| 功能 | 是否适用于普通 object | 说明 |
| --- | --- | --- |
| 查看并重命名物体 | 是 | 点击标题弹窗修改 `name` / `displayName` |
| 编辑位置 X/Y/Z | 是 | 数字滑杆 + 文本输入，写入 `x/y/z` |
| 编辑 Box 宽高深 | 是，仅 `model === 'box'` | 独立控制 `width/height/depth` |
| 编辑统一尺寸 Size | 是，仅 `model !== 'box'` | 同步写入 `width/height/depth` |
| 编辑三轴旋转 | 是 | UI 角度输入，内部弧度存储 |
| 主场景紫色圆环旋转 | 是 | 选中物体后显示 Three.js TransformControls，拖动后写回 `rotation` |
| 编辑 Tint Color | 是 | 写入 `tintColor` 并影响材质 emissive |
| Drop Object | 是 | 将物体下落到地面/角色/其它物体表面 |
| 搜索并切换内置模型 | 是 | Model Tab 中按名称/关键词搜索，点击缩略图切换 |
| 导入自定义模型 | 是 | 选择本地文件，复制后写入 `model` |
| 多选编辑 | 不是本文重点 | 多选会进入 `MultiSelectionInspector`，不展示单物体完整属性 Tab |
