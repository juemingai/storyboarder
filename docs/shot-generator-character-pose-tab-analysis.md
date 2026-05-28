# Shot Generator 人物编辑状态 Pose（姿势）Tab 功能与技术实现分析

## 1. 范围说明

本文档分析 Shot Generator 中选中人物（`sceneObject.type === 'character'`）后，属性面板里的 **Pose / 姿势 Tab** 的功能、UI 设计、数据流和代码文件位置。

截图中红色箭头指向的是人物属性面板的“姿势”标签页：上方是人物名称和 `特性/Properties` 标题，下方一排 Tab 图标中，Pose Tab 被激活后展示姿势搜索、保存当前姿势、镜像姿势和姿势预设缩略图网格。

> 说明：截图里面板位于界面左侧，但从功能语义上属于选中人物后的属性 Inspector 区域。本文档统一称为“人物属性面板 Pose Tab”。

## 2. 入口与显示条件

Pose Tab 由 `InspectedElement` 统一组织。只有当当前选中对象类型是 `character` 时才会显示。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/index.js` | 根据选中对象类型决定显示哪些 Tab；Character 才显示 Pose Tab |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | Pose Tab 主组件，实现搜索、保存、镜像、列表渲染 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | 单个姿势预设缩略图项，负责选择姿势、生成缩略图、删除入口 |

`InspectedElement` 中与 Pose Tab 相关的逻辑：

```js
const charPoseTab = useMemo(() => {
  if (!isChar(selectedType)) return nullTab

  return {
    tab: <Tab><Icon src='icon-tab-pose'/></Tab>,
    panel: <Panel><PosePresetsInspector/></Panel>
  }
}, [selectedType])
```

Tab 排列顺序：

1. 参数 Tab：`GeneralInspector`
2. 手部姿势 Tab：`HandInspector`
3. 姿势 Tab：`PosePresetsInspector`
4. 模型 Tab：`ModelInspector`
5. 附件 Tab：`AttachableInspector`
6. 表情 Tab：`EmotionInspector`
7. 头发 Tab：`HairInspector`

## 3. Pose Tab UI 结构

Pose Tab 内容由 `PosePresetsInspector/index.js` 渲染，主要分为四个区域：

| UI 区域 | 说明 | 关键组件/样式 |
| --- | --- | --- |
| 搜索栏 | 输入关键词过滤姿势预设 | `SearchList`、`.thumbnail-search input` |
| `+` 按钮 | 将当前人物骨骼姿势保存为新预设 | `.button_add`、`Modal`、`.modalInput` |
| `Mirror Pose` 按钮 | 左右镜像当前人物姿势 | `.mirror_button__wrapper`、`.mirror_button` |
| 姿势缩略图网格 | 展示并选择姿势预设；自定义预设可删除 | `Scrollable`、`Grid`、`PosePresetInspectorItem`、`.thumbnail-search__item` |

整体容器使用：

```jsx
<div className="thumbnail-search column">
  ...搜索栏和添加按钮...
  ...Mirror Pose...
  <Scrollable>
    <Grid ... />
  </Scrollable>
</div>
```

### 3.1 搜索栏 UI

搜索栏位于 Pose Tab 顶部，placeholder 使用 i18n key：

```js
t("shot-generator.inspector.pose-preset.search-pose")
```

实现方式：

- `PosePresetsInspector` 将所有姿势预设转换成 `{ value, id }` 搜索项。
- `value` 由 `preset.name + "|" + preset.keywords` 组成。
- `SearchList` 使用 `liquidmetal` 做模糊匹配。
- 匹配阈值为 `LiquidMetal.score(...) > 0.8`。
- 搜索结果通过 `saveFilteredPresets` 写入组件本地状态 `results`。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | 生成搜索列表、保存过滤后的 `results` |
| `src/js/shot-generator/components/SearchList/index.js` | 搜索输入、模糊匹配、阻止键盘事件冒泡 |
| `src/js/shot-generator/utils/searchPresetsForTerms.js` | 预设排序函数 `comparePresetNames` / `comparePresetPriority` |

### 3.2 `+` 保存姿势按钮

搜索栏右侧的 `+` 按钮用于把当前人物的骨骼状态保存为一个新的 Pose Preset。

交互流程：

1. 点击 `+` 按钮触发 `onCreatePosePreset`。
2. 生成默认名称：`Pose ${shortId(UUID)}`。
3. 打开 `Modal`。
4. 用户输入名称。
5. 点击 Add Preset 按钮触发 `addNewPosePreset(name)`。
6. 从 Redux 中读取当前人物最新 `sceneObject.skeleton`。
7. 生成新预设对象并 dispatch `createPosePreset(newPreset)`。
8. 调用 `updateObject(id, { posePresetId: newPreset.id })` 选中新预设。
9. 将用户自定义姿势预设保存到本地 `presetsStorage`。
10. 同时通过 `request.post` 上报到 `https://storyboarders.com/api/create_pose`。

新预设数据结构：

```js
{
  id: THREE.Math.generateUUID(),
  name,
  keywords: name,
  state: {
    skeleton: skeleton || {}
  },
  priority: 0
}
```

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | `onCreatePosePreset`、`addNewPosePreset`、Modal UI |
| `src/js/shared/reducers/shot-generator.js` | `CREATE_POSE_PRESET`、`createPosePreset`、`UPDATE_OBJECT` |
| `src/js/shared/store/presetsStorage.js` | 将用户自定义姿势保存到本地预设文件 |
| `src/js/shot-generator/components/Modal/index.js` | 弹窗容器 |
| `src/css/shot-generator.css` | `.button_add`、`.skeleton-selector__div`、`.skeleton-selector__button`、`.modalInput` |

### 3.3 Mirror Pose 镜像姿势按钮

`Mirror Pose` 按钮用于将当前人物姿势左右镜像。

实现逻辑：

1. 读取当前人物 `sceneObject.skeleton`。
2. 遍历每个骨骼 key。
3. 将骨骼 rotation 转成 quaternion。
4. 对 quaternion 的 `x` 和 `w` 取反。
5. 如果骨骼名包含 `Left` 或 `Right`，互换左右名称。
6. 将镜像后的 quaternion 转回 Euler rotation。
7. 组装 `oppositeSkeleton` 数组。
8. 调用 `updateCharacterIkSkeleton({ id, skeleton: oppositeSkeleton })` 批量写回人物骨骼。

核心代码位置：

```js
const mirrorSkeleton = () => {
  ...
  let mirroredQuat = new THREE.Quaternion().setFromEuler(
    new THREE.Euler(boneRot.x, boneRot.y, boneRot.z)
  )
  mirroredQuat.x *= -1
  mirroredQuat.w *= -1
  ...
  updateCharacterIkSkeleton({ id: sceneObject.id, skeleton: oppositeSkeleton })
}
```

注意点：

- 该功能依赖骨骼命名中包含 `Left` / `Right`。
- `position` 会从原 skeleton 中带入镜像后的骨骼记录。
- 更新使用 `UPDATE_CHARACTER_IK_SKELETON`，因此可以一次写入多个 bone。
- 写入后 `checksReducer` 会检测 skeleton 与当前 `posePresetId` 是否仍一致；如果不一致，会清除当前 preset 引用。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | `mirrorSkeleton`、`limbsSide`、Mirror Pose 按钮 UI |
| `src/js/shared/reducers/shot-generator.js` | `UPDATE_CHARACTER_IK_SKELETON` 批量更新骨骼 |
| `src/css/shot-generator.css` | `.mirror_button__wrapper`、`.mirror_button` |

### 3.4 姿势预设缩略图网格

姿势列表使用虚拟化/延迟渲染思路的 `Grid` 组件展示。每个预设由 `PosePresetInspectorItem` 渲染。

功能点：

- 每个 item 显示姿势缩略图和姿势名称。
- 当前选中姿势通过 `.thumbnail-search__item--selected` 高亮。
- 点击 item 会应用该姿势。
- 内置姿势不可删除。
- 用户自定义姿势 hover 时显示 `X` 删除入口。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | 将 `results` 传给 `Grid`，并提供 itemData |
| `src/js/shot-generator/components/Grid/index.js` | 按行列渲染列表，滚动时逐步渲染更多项 |
| `src/js/shot-generator/components/Scrollable/index.js` | 给列表区域提供滚动容器 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | 姿势 item UI、选择、缩略图生成、删除入口 |
| `src/js/shot-generator/components/InspectedElement/RemovableItem/RemovableItem.js` | hover 显示删除按钮，阻止删除事件冒泡 |
| `src/js/shot-generator/utils/InspectorElementsSettings.js` | `NUM_COLS`、`ITEM_HEIGHT`、`ITEM_WIDTH`、`IMAGE_HEIGHT`、`IMAGE_WIDTH` 等尺寸配置 |

### 3.5 选择姿势并应用到人物

点击一个姿势预设时，`PosePresetInspectorItem` 会执行：

```js
undoGroupStart()
updateObject(id, { posePresetId, skeleton })
setTimeout(() => {
  undoGroupEnd()
}, 50)
```

其中：

- `id` 是当前选中的 character id。
- `posePresetId` 是预设 id。
- `skeleton` 是预设中的 `preset.state.skeleton`。
- `undoGroupStart` / `undoGroupEnd` 将应用姿势作为一个撤销步骤。

数据更新后：

1. Redux `UPDATE_OBJECT` 更新人物对象的 `posePresetId` 和 `skeleton`。
2. `Character` 组件收到新的 `sceneObject.skeleton`。
3. Three.js SkinnedMesh 骨骼按 skeleton rotation 更新。
4. 主画布中人物姿势变化。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | 点击姿势并调用 `updateObject(id, { posePresetId, skeleton })` |
| `src/js/shared/reducers/shot-generator.js` | `UPDATE_OBJECT` 写入人物对象数据 |
| `src/js/shot-generator/components/Three/Character.js` | 根据 `sceneObject.skeleton` 更新人物模型骨骼 |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | 向 `Character` 注入 `updateCharacterSkeleton`、`updateCharacterIkSkeleton` 等能力 |

## 4. 姿势缩略图生成机制

`PosePresetInspectorItem` 会为每个姿势生成并缓存一张 jpg 缩略图。

缩略图路径：

```js
path.join(remote.app.getPath('userData'), 'presets', 'poses', `${preset.id}.jpg`)
```

生成条件：

- 如果该 jpg 文件不存在，则使用 `ThumbnailRenderer` 临时渲染。
- 如果文件已存在，则直接用缓存图。

生成流程：

1. `PosePresetsInspector` 通过 `useAsset(characterPath)` 加载默认人物模型资源。
2. `PosePresetInspectorItem` 检查缩略图文件是否存在。
3. 不存在时创建 `ThumbnailRenderer`。
4. `createCharacter(gltf)` 克隆人物 glTF，取 SkinnedMesh 和 armature。
5. `setupRenderer` 将姿势 preset 的 skeleton rotation 应用到缩略图人物骨骼。
6. `thumbnailRenderer.current.render()` 渲染离屏图。
7. `toDataURL('image/jpg')` 输出 base64。
8. 写入 `userData/presets/poses/{preset.id}.jpg`。
9. item 中 `<Image src={src} className="thumbnail" />` 显示缩略图。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | 通过 `useAsset(characterPath)` 加载缩略图用人物模型 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | `createCharacter`、`setupRenderer`、缓存缩略图文件 |
| `src/js/shot-generator/utils/ThumbnailRenderer.js` | 离屏渲染缩略图 |
| `src/js/shot-generator/helpers/cloneGltf.js` | 克隆 glTF，避免直接修改原资源 |
| `src/js/shot-generator/helpers/outlineMaterial.js` | `patchMaterial` 处理缩略图材质描边效果 |
| `src/js/shot-generator/components/Image/index.js` | 图片加载显示组件 |

## 5. 删除自定义姿势预设

用户自定义姿势可以删除，内置姿势不能删除。

判断是否内置：

```js
const isDefaultPreset = (id) => {
  return defaultArray.find(image => image.id === id)
}
```

删除流程：

1. `PosePresetInspectorItem` 把 `isRemovable={!isDefaultPreset(preset.id)}` 传给 `RemovableItem`。
2. 鼠标 hover 自定义预设时显示 `X`。
3. 点击 `X` 后 `RemovableItem` 调用 `onRemoval(data)`。
4. 弹出确认框。
5. 确认后更新本地 presetsStorage，过滤掉被删除的用户预设。
6. dispatch `deletePosePreset(data.id)`。
7. reducer 执行 `DELETE_POSE_PRESET`，从 `state.presets.poses` 删除该项。

相关代码：

| 文件 | 作用 |
| --- | --- |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js` | 判断内置/自定义，传入 `isRemovable` |
| `src/js/shot-generator/components/InspectedElement/RemovableItem/RemovableItem.js` | hover 显示 `X`，点击删除并阻止冒泡 |
| `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js` | `onRemoval` 确认删除、保存本地预设、dispatch 删除 action |
| `src/js/shared/reducers/shot-generator.js` | `DELETE_POSE_PRESET`、`deletePosePreset` |
| `src/js/shared/store/presetsStorage.js` | 保存删除后的用户姿势集合 |

## 6. 数据来源与状态结构

Pose Tab 读取的数据来自 Redux：

| Redux 数据 | 用途 |
| --- | --- |
| `state.presets.poses` | 所有姿势预设，包含内置和用户自定义 |
| `getSelections(state)[0]` | 当前选中的人物 id |
| `getSceneObjects(state)[id]` | 当前人物对象，用于读取 skeleton、posePresetId、model |
| `state.attachments[filepath]` | 默认人物模型资源加载状态，用于缩略图渲染 |

人物对象中与 Pose 相关的字段：

| 字段 | 说明 |
| --- | --- |
| `posePresetId` | 当前套用的姿势预设 id；当骨骼被手动改动且不再匹配预设时会失效 |
| `skeleton` | 当前人物骨骼姿势，按 bone name 存 rotation / position / quaternion 等 |
| `handPosePresetId` | 手部姿势预设 id；设置全身姿势时可能清空手部预设 |
| `handSkeleton` | 手部骨骼数据；部分骨骼更新会同步影响手部骨骼 |

内置姿势来源：

- `src/js/shared/reducers/shot-generator-presets/poses.json`

用户姿势持久化：

- 通过 `presetsStorage.savePosePresets({ poses })` 保存。
- 缩略图缓存写到 Electron `userData/presets/poses/*.jpg`。

## 7. Redux 与骨骼更新逻辑

### 7.1 应用预设

点击预设时走普通对象更新：

```text
PosePresetInspectorItem.onPointerDown
  -> updateObject(id, { posePresetId, skeleton })
  -> UPDATE_OBJECT
  -> sceneObjects[id].posePresetId / skeleton 更新
  -> Character 组件重渲染并应用骨骼
```

### 7.2 镜像或 IK 批量更新

镜像姿势走 IK skeleton 批量更新：

```text
mirrorSkeleton
  -> updateCharacterIkSkeleton({ id, skeleton: oppositeSkeleton })
  -> UPDATE_CHARACTER_IK_SKELETON
  -> 按 bone.name 更新 sceneObjects[id].skeleton
  -> checksReducer 检测是否仍匹配 posePresetId
```

### 7.3 单骨骼编辑关联

虽然不属于 Pose Tab 主 UI，但人物姿势可通过骨骼控制器或 BoneInspector 手动改变。相关更新：

| Action | 用途 |
| --- | --- |
| `UPDATE_CHARACTER_SKELETON` | 按骨骼名更新单个 bone rotation |
| `UPDATE_CHARACTER_IK_SKELETON` | 批量更新多个 bone rotation / position / quaternion |

相关文件：

- `src/js/shared/reducers/shot-generator.js`
- `src/js/shot-generator/components/Three/Character.js`
- `src/js/shot-generator/SceneManagerR3fLarge.js`
- `src/js/shared/IK/IkHelper.js`
- `src/js/shared/IK/objects/IkObjects/Ragdoll.js`

## 8. UI 样式设计说明

Pose Tab 沿用 Shot Generator 的暗色 Inspector 视觉系统：

| 设计项 | 实现 |
| --- | --- |
| 面板背景 | `#inspector` 使用深灰背景 `#333639` |
| Tab 图标 | `Tabs` + `Icon`，激活态通过 `.tabs-tab.active` 表达 |
| 搜索框 | 深色输入框、圆角、focus 时提高背景亮度 |
| 添加按钮 | 小型方形 `+` 按钮，与搜索框同一行 |
| 镜像按钮 | 横向满宽按钮，低对比深色背景，hover 提升亮度 |
| 缩略图网格 | 每项上方图片，下方名称；选中后文字变白 |
| 删除入口 | 自定义预设 hover 时显示 `X`，内置预设不显示 |
| 滚动 | `Scrollable` 包裹 `Grid`，隐藏 WebKit scrollbar |

主要 CSS 选择器：

| 选择器 | 作用 |
| --- | --- |
| `#inspector` | 属性面板容器 |
| `.tabs-header` / `.tabs-tab` / `.tabs-body__content` | 标签页布局与激活状态 |
| `.thumbnail-search` | Pose/Model/Emotion 等缩略图型面板通用容器 |
| `.thumbnail-search input` | 搜索框 |
| `.thumbnail-search__item` | 缩略图 item |
| `.thumbnail-search__item--selected` | 当前选中预设 |
| `.thumbnail-search__name` | item 名称文字 |
| `#inspector .button_add` | `+` 添加按钮 |
| `.mirror_button__wrapper` / `.mirror_button` | Mirror Pose 按钮 |
| `.skeleton-selector__div` / `.skeleton-selector__button` | Modal 内确认按钮区域 |
| `.modalInput` | Modal 内预设名称输入框 |

样式文件：

- `src/css/shot-generator.css`

## 9. 文件清单

### 9.1 Pose Tab 主链路

- `src/js/shot-generator/components/InspectedElement/index.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js`
- `src/js/shot-generator/components/InspectedElement/RemovableItem/RemovableItem.js`

### 9.2 通用 UI 组件

- `src/js/shot-generator/components/Tabs/index.js`
- `src/js/shot-generator/components/SearchList/index.js`
- `src/js/shot-generator/components/Grid/index.js`
- `src/js/shot-generator/components/Scrollable/index.js`
- `src/js/shot-generator/components/Modal/index.js`
- `src/js/shot-generator/components/Image/index.js`
- `src/js/shot-generator/components/Icon/index.js`

### 9.3 状态、预设和持久化

- `src/js/shared/reducers/shot-generator.js`
- `src/js/shared/reducers/shot-generator-presets/poses.json`
- `src/js/shared/store/presetsStorage.js`
- `src/js/shot-generator/utils/searchPresetsForTerms.js`
- `src/js/shot-generator/utils/InspectorElementsSettings.js`

### 9.4 缩略图与 3D 骨骼应用

- `src/js/shot-generator/utils/ThumbnailRenderer.js`
- `src/js/shot-generator/helpers/cloneGltf.js`
- `src/js/shot-generator/helpers/outlineMaterial.js`
- `src/js/shot-generator/components/Three/Character.js`
- `src/js/shot-generator/SceneManagerR3fLarge.js`
- `src/js/shared/IK/IkHelper.js`
- `src/js/shared/IK/objects/IkObjects/Ragdoll.js`

### 9.5 样式

- `src/css/shot-generator.css`

## 10. 功能总结

人物 Pose Tab 提供以下能力：

| 功能 | 说明 |
| --- | --- |
| 搜索姿势 | 按名称和关键词模糊过滤 `state.presets.poses` |
| 应用姿势预设 | 点击缩略图后写入 `posePresetId` 和 `skeleton` |
| 保存当前姿势 | 点击 `+`，命名后将当前 skeleton 保存为用户预设 |
| 镜像当前姿势 | 对骨骼左右名称和 quaternion 做镜像转换后批量更新 skeleton |
| 缩略图缓存 | 首次展示时离屏渲染 jpg，之后复用 `userData/presets/poses/*.jpg` |
| 删除用户姿势 | 自定义预设 hover 显示 `X`，确认后删除并保存本地预设集合 |
| 撤销支持 | 应用姿势和保存姿势时使用 `undoGroupStart` / `undoGroupEnd` 分组 |
