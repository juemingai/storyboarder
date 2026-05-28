# Shot Generator 角色选中后的左侧属性面板功能移植分析

本文档说明 Storyboarder Shot Generator 中“点击人物角色后，左侧菜单 / 属性面板包含哪些功能、如何实现、数据如何写回”，并给出移植到 FlowCanvas 的建议结构。

截图中左侧区域可以分成两层：

- **Scene 对象列表**：显示相机、人物、物体等场景对象，支持选择、锁定、显隐、删除。
- **Character 属性面板**：选中人物后显示 `Default Character 属性`，包含多个 tab：基础参数、手势、姿势、模型、挂件、表情、头发。

## 1. 入口与组件层级

左侧整体入口：

- `src/js/shot-generator/components/ElementsPanel/index.js`

结构：

```jsx
<div style={{ flex: 1, display: "flex", flexDirection: "column" }}>
  <div id="listing">
    <ItemList/>
  </div>
  <Inspector
    world={world}
    kind={kind}
    data={data}
    models={models}
    updateObject={updateObject}
    selectedBone={selectedBone}
    selectBone={selectBone}
    updateCharacterSkeleton={updateCharacterSkeleton}
    storyboarderFilePath={storyboarderFilePath}
    selections={selections}
  />
</div>
```

数据来源：

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

主要 action：

```text
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
undoGroupStart
undoGroupEnd
```

## 2. Scene 对象列表

对象列表实现文件：

- `src/js/shot-generator/components/ItemList/index.js`
- `src/js/shot-generator/components/ItemList/Item.js`

截图上方列表中的：

```text
Scene
Camera 1
Default Character
```

就是 `ItemList` 渲染的内容。

### 2.1 排序规则

对象按固定类型顺序显示：

```js
const sortPriority = ['camera', 'character', 'object', 'image', 'light', 'volume', 'group']
```

列表会先渲染 `Scene`，然后渲染排序后的 `sceneObjects`。

### 2.2 选择对象

点击列表项会调用：

```js
selectObject([...new Set(currentSelections)])
```

如果点击的是 camera，还会设置 active camera：

```js
if(props.type === "camera") {
  setActiveCamera(props.id)
}
```

支持 `Shift` 多选：

```js
if (event.shiftKey) {
  // 在已有 selections 中加入或移除当前对象
}
```

### 2.3 锁定 / 解锁

锁图标由 `Item.js` 渲染：

```jsx
<Icon src={props.locked ? 'icon-item-lock' : 'icon-item-unlock'}/>
```

点击后：

```js
let nextAvailability = !props.locked
updateObject(props.id, {locked: nextAvailability})
```

如果对象是 group，也会同步更新 children。

### 2.4 显示 / 隐藏

眼睛图标由 `Item.js` 渲染：

```jsx
<Icon src={props.visible ? 'icon-item-visible' : 'icon-item-hidden'}/>
```

点击后：

```js
let nextVisibility = !props.visible
updateObject(props.id, {visible: nextVisibility})
```

相机对象不显示 eye 图标。

### 2.5 删除对象

删除按钮是右侧的 `X`：

```jsx
<a className="delete">X</a>
```

删除前会弹确认框：

```js
dialog.showMessageBox(null, {
  type: 'question',
  buttons: ['Yes', 'No'],
  message: 'Are you sure?',
  defaultId: 1
})
```

删除人物时，会连同挂在人物身上的 attachables 一起删除：

```js
if(props.type === "character") {
  let attachableIds = Object.values(sceneObjects)
    .filter(obj => obj.attachToId === props.id)
    .map(obj => obj.id)
  idsToRemove = attachableIds.concat(idsToRemove)
}
deleteObjects(idsToRemove)
```

## 3. Character 属性面板总览

选中角色后，下方会显示：

```text
Default Character 属性:
```

实现文件：

- `src/js/shot-generator/components/InspectedElement/index.js`

该组件根据当前选中对象类型决定显示哪些 tab。

角色类型会显示 7 个 tab：

| Tab | 图标 key | 功能 | 组件 |
|---|---|---|---|
| 基础参数 | `icon-tab-parameters` | 位置、旋转、高度、头部比例、颜色、角色预设、骨骼数值 | `GeneralInspector` |
| 手势 | `icon-tab-hand` | 左手 / 右手 / 双手手势预设 | `HandInspector` |
| 姿势 | `icon-tab-pose` | 全身姿势预设、保存姿势、镜像姿势 | `PosePresetsInspector` |
| 模型 | `icon-tab-model` | 切换人物模型、导入自定义 GLB | `ModelInspector` |
| 挂件 | `icon-tab-attachable` | 给骨骼绑定道具 / 配件 | `AttachableInspector` |
| 表情 | `icon-tab-emotions` | 选择或导入脸部表情贴图 | `EmotionInspector` |
| 头发 | `icon-tab-hair` | 选择 / 删除 / 导入头发模型 | `HairInspector` |

对应代码：

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

Tab 状态由自定义组件管理：

- `src/js/shot-generator/components/Tabs/index.js`

`Tabs` 内部维护：

```js
useState({ current: 0, prev: 0 })
```

点击 tab 时更新 `current`，`Panel` 只渲染当前 active 的内容。

## 4. 标题与重命名

标题实现位置：

- `src/js/shot-generator/components/InspectedElement/index.js`

标题显示：

```jsx
{selectedName} {t("shot-generator.inspector.inspected-element.properties")}
```

点击标题会打开 modal，用于修改角色名称：

```js
updateObject(id, {
  displayName: changedName,
  name: changedName
})
```

FlowCanvas 移植时建议保留这个交互：点击对象名可以重命名当前角色。

## 5. 基础参数 tab

截图红框中的左侧参数区域对应第一个 tab：

- `src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/GeneralInspector/Character.js`

`GeneralInspector` 根据 `sceneObject.type` 选择具体 inspector：

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

角色使用 `CharacterInspector`。

### 5.1 角色预设

最上方 `预设 Default Character +` 来自：

- `src/js/shot-generator/components/InspectedElement/CharacterPresetEditor/index.js`

功能：

- 下拉选择已有 character preset。
- 点击 `+` 将当前角色外观保存为新 preset。
- 应用 preset 时更新模型、高度、头部比例、肤色、体型 morph，并重置姿势。
- 如果 preset 带 attachables，会重新创建挂件。

选择 preset 时：

```js
dispatch(updateObject(sceneObject.id, {
  characterPresetId,
  height: preset.state.height,
  model: preset.state.model,
  headScale: preset.state.headScale,
  tintColor: preset.state.tintColor,
  rotation: 0,
  morphTargets: {
    mesomorphic: preset.state.morphTargets.mesomorphic,
    ectomorphic: preset.state.morphTargets.ectomorphic,
    endomorphic: preset.state.morphTargets.endomorphic
  },
  name: sceneObject.name || preset.name,
  posePresetId: defaultCharacterPreset.id,
  skeleton: defaultCharacterPreset.state.skeleton
}))
```

创建 preset 时保存：

```js
{
  height,
  model,
  headScale,
  tintColor,
  morphTargets
}
```

如果当前人物有 attachables，还会把 attachables 一起写入 preset。

### 5.2 位置 X / Y / Z

截图中的：

```text
X
Y
Z
```

对应：

```jsx
<NumberSlider label="X" value={props.x} min={-30} max={30} onSetValue={setX}/>
<NumberSlider label="Y" value={props.y} min={-30} max={30} onSetValue={setY}/>
<NumberSlider label="Z" value={props.z} min={-30} max={30} onSetValue={setZ}/>
```

写回：

```js
updateObject(id, { x })
updateObject(id, { y })
updateObject(id, { z })
```

注意坐标系：

- 状态中的 `x/y/z` 是 Shot Generator 自己的导演台坐标。
- Three.js 渲染中角色位置映射为 `position={[x, z, y]}`。
- FlowCanvas 移植时应集中处理状态坐标和 Three.js 坐标转换。

### 5.3 旋转

截图中的 `旋转` 对应：

```jsx
<NumberSlider
  label={t("shot-generator.inspector.common.rotation")}
  value={_Math.radToDeg(props.rotation)}
  min={-180}
  max={180}
  step={1}
  onSetValue={setRotation}
  transform={transforms.degrees}
  formatter={formatters.degrees}
/>
```

写回时将 degree 转 rad：

```js
updateObject(id, { rotation: _Math.degToRad(x) })
```

### 5.4 高度 / 缩放

内置角色显示 `高度`：

```jsx
<NumberSlider
  label={t("shot-generator.inspector.common.height")}
  value={props.height}
  min={heightRange.min}
  max={heightRange.max}
  step={0.0254}
  onSetValue={setHeight}
/>
```

自定义模型显示 `scale`：

```jsx
<NumberSlider
  label={t("shot-generator.inspector.common.scale")}
  min={0.3}
  max={3.05}
  step={0.0254}
  value={sceneObject.height}
  onSetValue={setHeight}
/>
```

高度范围根据角色模型类型决定：

```js
const CHARACTER_HEIGHT_RANGE = {
  character: { min: 1.4732, max: 2.1336 },
  child: { min: 1.003, max: 1.384 },
  baby: { min: 0.492, max: 0.94 }
}
```

显示格式支持英尺英寸：

```js
feetAndInchesAsString(...metersAsFeetAndInches(sceneObject.height))
```

### 5.5 头部比例

内置角色显示 `头部`：

```jsx
<NumberSlider
  label={t("shot-generator.inspector.character.head")}
  value={props.headScale * 100}
  min={80}
  max={120}
  step={1}
  formatter={formatters.percent}
  onSetValue={setHeadScale}
/>
```

写回：

```js
updateObject(id, { headScale: value / 100 })
```

`Character.js` 中会把它应用到 `Head` bone：

```js
headBone.scale.setScalar(sceneObject.headScale)
```

### 5.6 肤色 / tintColor

内置角色显示 `淡色` / tint color：

```jsx
<ColorSelect
  label={t("shot-generator.inspector.common.tint-color")}
  value={props.tintColor}
  onSetValue={setTintColor}
/>
```

写回：

```js
updateObject(id, { tintColor })
```

`Character.js` 中会应用到材质 emissive：

```js
lod.children.forEach(skinnedMesh => {
  skinnedMesh.material.emissive.set(sceneObject.tintColor)
})
```

### 5.7 Drop Character

基础 tab 下方有 `drop-character` 按钮：

```jsx
<div className="drop_button" onClick={() => ipcRenderer.send('shot-generator:object:drops')}>
  {t("shot-generator.inspector.character.drop-character")}
</div>
```

它通过 IPC 触发：

```text
shot-generator:object:drops
```

用于将角色落到可放置表面 / 地面等位置。FlowCanvas 如果移植，应实现为“角色落地 / 对齐地面”操作。

### 5.8 体型 morphs

内置角色如果有 `validMorphTargets`，会显示体型滑杆：

```js
const MORPH_TARGET_LABELS = {
  mesomorphic: 'muscular',
  ectomorphic: 'skinny',
  endomorphic: 'obese',
}
```

写回：

```js
updateObject(id, {
  morphTargets: { [key]: value / 100 }
})
```

`Character.js` 中应用到 mesh 的 morph target：

```js
modelSettings.validMorphTargets.forEach((name, index) => {
  skinnedMesh.morphTargetInfluences[index] = sceneObject.morphTargets[name]
})
```

### 5.9 骨骼数值编辑

如果当前选中了某根 bone，基础 tab 下方会额外显示：

- `src/js/shot-generator/components/InspectedElement/BoneInspector/index.js`

功能：

- 显示 bone name。
- 显示 x / y / z 三个角度 slider。
- 修改后调用 `updateCharacterSkeleton`。

```js
updateCharacterSkeleton({
  id: sceneObject.id,
  name: bone.name,
  rotation: {
    x: bone.rotation.x,
    y: bone.rotation.y,
    z: bone.rotation.z,
    [key]: THREE.Math.degToRad(value)
  }
})
```

## 6. 手势 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/index.js`
- `src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/HandPresetsEditorItem.js`
- `src/js/utils/handSkeletonUtils.js`

功能：

- 搜索手势预设。
- 点击 `+` 将当前手势保存为新 preset。
- 选择保存左手或右手。
- 选择应用到左手、右手或双手。
- 如果应用到对侧手，会使用手势镜像算法。

状态字段：

```text
sceneObject.handPosePresetId
sceneObject.handSkeleton
```

应用 preset：

```js
updateObject(id, { handPosePresetId, handSkeleton })
```

对侧镜像工具：

```js
createdMirroredHand(handSkeleton, presetHand)
applyChangesToSkeleton(currentSkeleton, oppositeSkeleton)
getOppositeHandName(presetHand)
```

## 7. 姿势 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/PosePresetsInspector/PosePresetInspectorItem.js`

功能：

- 搜索全身姿势预设。
- 点击 `+` 将当前角色 skeleton 保存为新姿势。
- 点击姿势缩略图应用全身姿势。
- 支持删除用户自定义姿势。
- 支持 `镜像姿势`。

状态字段：

```text
sceneObject.posePresetId
sceneObject.skeleton
```

应用姿势：

```js
updateObject(id, {
  posePresetId,
  skeleton: preset.state.skeleton
})
```

镜像姿势：

```js
updateCharacterIkSkeleton({
  id: sceneObject.id,
  skeleton: oppositeSkeleton
})
```

该功能已在 `docs/shot-generator-mirror-pose-migration.md` 单独分析。

## 8. 模型 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/ModelInspector/index.js`

功能：

- 搜索可用人物模型。
- 网格显示模型缩略图。
- 点击模型切换 `sceneObject.model`。
- 通过 `FileInput` 导入自定义模型。
- 切换模型时根据内置 / 自定义模型差异重置 skeleton。

切换内置模型：

```js
updateObject(sceneObject.id, { model: currentModel })
```

从自定义模型切回内置模型时，会恢复默认 skeleton：

```js
let defaultSkeleton = getDefaultPosePreset().state.skeleton
updateCharacterIkSkeleton({ id: sceneObject.id, skeleton: [] })
updateCharacterIkSkeleton({ id: sceneObject.id, skeleton })
```

从内置切换到自定义，或自定义切换自定义，会清空 skeleton：

```js
updateCharacterIkSkeleton({ id: sceneObject.id, skeleton: [] })
```

导入自定义模型：

```js
let updatedModel = CopyFile(storyboarderFilePath, filepath.file, sceneObject.type)
resetSkeleton(updatedModel)
```

## 9. 挂件 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/AttachableEditor/index.js`
- `src/js/shot-generator/components/InspectedElement/HandInspector/HandSelectionModal/index.js`

功能：

- 搜索可挂载模型。
- 导入自定义 attachable。
- 选择模型后绑定到指定 bone。
- 显示当前角色已有 attachables。
- 修改 attachable 绑定骨骼。
- 修改 attachable size。
- 删除 attachable。

创建 attachable：

```js
let element = {
  id: THREE.Math.generateUUID(),
  type: 'attachable',
  x: modelData.x,
  y: modelData.y,
  z: modelData.z,
  model: modelData.id,
  name: modelData.name,
  bindBone: name || modelData.bindBone,
  attachToId: id,
  size: 1,
  status: "PENDING",
  rotation: modelData.rotation
}
createObject(element)
selectAttachable({ id: element.id, bindId: element.attachToId })
```

编辑 attachable bone：

```js
updateObject(id, { bindBone: name })
```

编辑 size：

```js
updateObject(sceneObject.id, { size: value })
```

## 10. 表情 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js`
- `src/js/shot-generator/components/InspectedElement/EmotionInspector/thumbnail-renderer.js`

功能：

- 显示系统表情和用户表情。
- 搜索表情。
- 选择表情贴图。
- 导入图片创建用户表情。
- 生成表情缩略图。
- 删除用户表情。

状态字段：

```text
sceneObject.emotionPresetId
```

选择表情：

```js
dispatch(updateObject(sceneObjectId, { emotionPresetId }))
```

取消表情：

```js
dispatch(updateObject(sceneObjectId, { emotionPresetId: undefined }))
```

导入用户表情时：

- 拷贝图片到用户 presets 目录。
- 创建 emotion preset。
- 写入 `presetsStorage.saveEmotionsPresets`。
- 自动选择新表情。

`Character.js` 中根据 `emotionPresetId` 加载表情 texture，并交给 `FaceMesh` 绘制。

## 11. 头发 tab

实现文件：

- `src/js/shot-generator/components/InspectedElement/HairInspector/index.js`

头发本质上是特殊 attachable：

```js
{
  type: 'attachable',
  attachableType: 'hair',
  bindBone: 'Head'
}
```

功能：

- 搜索头发模型。
- 选择“无”来移除当前头发。
- 选择内置头发模型。
- 导入自定义头发模型。
- 每次只保留一个 hair attachable。

选择头发：

```js
if (selectedHair) {
  deleteObjects([selectedHair.id])
  deselectAttachable()
}

createObject({
  ...createSceneObjectForAttachable(),
  model: data.model,
  attachToId: selectedSceneObject.id,
  ...data.sceneObjectOverrides
})
```

移除头发：

```js
deleteObjects([selectedHair.id])
deselectAttachable()
```

自定义头发会先拷贝到项目模型目录：

```js
let model = CopyFile(storyboarderFilePath, filepath.file, 'attachable')
```

## 12. FlowCanvas 移植建议

### 12.1 推荐模块拆分

```text
character-inspector/
├── CharacterInspectorPanel.tsx
├── CharacterInspectorTabs.tsx
├── tabs/
│   ├── CharacterGeneralTab.tsx
│   ├── CharacterHandPoseTab.tsx
│   ├── CharacterPosePresetTab.tsx
│   ├── CharacterModelTab.tsx
│   ├── CharacterAttachableTab.tsx
│   ├── CharacterEmotionTab.tsx
│   └── CharacterHairTab.tsx
├── components/
│   ├── CharacterPresetSelect.tsx
│   ├── BoneRotationInspector.tsx
│   ├── ModelGrid.tsx
│   ├── PresetGrid.tsx
│   └── AttachableList.tsx
└── state/
    ├── characterInspectorStore.ts
    ├── characterPresetStore.ts
    ├── posePresetStore.ts
    └── attachableStore.ts
```

### 12.2 推荐状态接口

```ts
interface CharacterObject {
  id: string
  type: 'character'
  name?: string
  displayName?: string
  model: string
  x: number
  y: number
  z: number
  rotation: number
  height: number
  headScale: number
  tintColor: string
  morphTargets: Record<string, number>
  skeleton: CharacterSkeletonPose
  handSkeleton?: CharacterSkeletonPose
  characterPresetId?: string | null
  posePresetId?: string | null
  handPosePresetId?: string | null
  emotionPresetId?: string | null
  visible: boolean
  locked?: boolean
}
```

### 12.3 推荐 action

```ts
updateCharacterTransform(id, patch)
updateCharacterAppearance(id, patch)
renameCharacter(id, name)
selectCharacterPreset(id, presetId)
saveCharacterPreset(id, name)
selectCharacterModel(id, modelId)
importCharacterModel(id, file)
selectPosePreset(id, presetId)
mirrorCharacterPose(id)
selectHandPosePreset(id, presetId, targetHand)
createAttachable(characterId, modelId, bindBone)
updateAttachable(id, patch)
deleteAttachable(id)
selectEmotionPreset(characterId, emotionPresetId)
selectHair(characterId, hairModelId)
dropCharacterToGround(id)
```

### 12.4 移植优先级

建议按以下顺序移植：

1. **对象列表**
   - 选择角色、锁定、显隐、删除。

2. **基础参数 tab**
   - x/y/z、rotation、height、headScale、tintColor。

3. **模型 tab**
   - 切换人物模型、重置 skeleton。

4. **姿势 tab**
   - 应用姿势预设、保存姿势、镜像姿势。

5. **手势 tab**
   - 应用手势预设、左右手镜像。

6. **挂件 / 头发**
   - 绑定到骨骼、跟随角色。

7. **表情**
   - 表情贴图、FaceMesh 或 FlowCanvas 等价实现。

## 13. 验收标准

### 对象列表

- 点击角色后角色被选中，属性面板切换为角色面板。
- 锁定角色后主视图不能拖动 / 编辑该角色。
- 隐藏角色后主视图不可见。
- 删除角色时自动删除挂在该角色上的 attachables。

### 基础参数

- 修改 X/Y/Z 后角色在主视图和俯视图同步移动。
- 修改旋转后角色朝向变化。
- 修改高度后角色等比缩放。
- 修改头部比例后只影响头部。
- 修改 tintColor 后角色材质颜色变化。
- 内置角色显示高度、头部、肤色、体型；自定义模型显示 scale 并隐藏不支持项。

### 预设与模型

- 选择 character preset 后模型、体型、肤色、头部比例更新。
- 保存 character preset 后可在列表中重新选择。
- 切换模型后 skeleton 按内置 / 自定义规则重置。
- 应用姿势 preset 后角色姿势变化。
- 镜像姿势后左右动作互换。

### 附加功能

- 手势 preset 可应用到左手、右手、双手。
- attachable 可绑定到指定 bone 并跟随角色。
- hair 始终绑定到 Head 且同一角色只保留一个。
- emotion preset 可改变角色脸部贴图。

## 14. 关键代码清单

```text
src/js/shot-generator/components/ElementsPanel/index.js
src/js/shot-generator/components/ItemList/index.js
src/js/shot-generator/components/ItemList/Item.js
src/js/shot-generator/components/InspectedElement/index.js
src/js/shot-generator/components/Tabs/index.js
src/js/shot-generator/components/InspectedElement/GeneralInspector/index.js
src/js/shot-generator/components/InspectedElement/GeneralInspector/Character.js
src/js/shot-generator/components/InspectedElement/CharacterPresetEditor/index.js
src/js/shot-generator/components/InspectedElement/BoneInspector/index.js
src/js/shot-generator/components/InspectedElement/HandInspector/HandPresetsEditor/index.js
src/js/shot-generator/components/InspectedElement/PosePresetsInspector/index.js
src/js/shot-generator/components/InspectedElement/ModelInspector/index.js
src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js
src/js/shot-generator/components/InspectedElement/AttachableEditor/index.js
src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js
src/js/shot-generator/components/InspectedElement/HairInspector/index.js
src/js/shared/reducers/shot-generator.js
```

如果 FlowCanvas 第一阶段只复刻截图红框中的基础功能，最少需要移植：

```text
GeneralInspector/Character.js
CharacterPresetEditor/index.js
ItemList/index.js
ItemList/Item.js
InspectedElement/index.js 中 character tab 框架
shot-generator.js 中 updateObject / selectObject / deleteObjects / visible / locked 相关 reducer
```
