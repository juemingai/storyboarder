# Shot Generator 情绪/表情 Tab 技术实现分析

本文分析 Shot Generator 人物编辑状态下，截图中“面具”图标的情绪/表情 Tab 如何工作，包括：预设列表展示、点击表情后如何应用到人物、用户自定义表情如何导入，以及最终表情图片如何绘制到人物脸部贴图上。

## 结论

这部分功能在代码中叫 `EmotionInspector`，数据上叫 `emotionPreset`。它并不是通过面部骨骼驱动表情，也不是通过 blendshape/morph target 切换表情；当前实现的核心是：

1. 情绪/表情预设本质是一张图片纹理：`<emotionPresetId>-texture.png`。
2. 点击某个表情卡片时，只是把当前人物的 `sceneObject.emotionPresetId` 更新为该预设 id。
3. 渲染人物时，`SceneManagerR3fLarge` 根据 `emotionPresetId` 解析出对应表情图片路径，作为 `imagePath` 传给 `Character`。
4. `Character` 加载该图片为 `THREE.Texture`，然后交给 `FaceMesh.draw(texture)`。
5. `FaceMesh` 会复制人物原始皮肤贴图到一个 canvas 上，再通过骨骼和 UV 计算找到脸部中心位置，把表情图片画到该 canvas 对应的脸部 UV 坐标上。
6. canvas 被包装成 `THREE.Texture` 并替换到人物材质的 `material.map`，所以视觉上就是表情图片贴到了脸上。

因此，这个 Tab 的“表情应用”是贴图合成方案，不是 3D 表情动画方案。

## 关键代码位置

- Tab 注册入口：`src/js/shot-generator/components/InspectedElement/index.js`
- 表情列表、搜索、选择、导入：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js`
- 表情缩略图生成：`src/js/shot-generator/components/InspectedElement/EmotionInspector/thumbnail-renderer.js`
- 表情绘制到脸部贴图：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js`
- 人物渲染与应用表情纹理：`src/js/shot-generator/components/Three/Character.js`
- 人物渲染时解析表情纹理路径：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 系统表情预设数据：`src/js/shared/reducers/shot-generator-presets/emotions.json`
- Redux reducer/actions：`src/js/shared/reducers/shot-generator.js`
- 路径服务：`src/js/shot-generator/services/filepaths.js`

## Tab 是如何挂进人物属性面板的

人物属性面板在 `InspectedElement/index.js` 中组合多个 Tab。只有当前选中对象是人物时才显示情绪/表情 Tab：

参考：`src/js/shot-generator/components/InspectedElement/index.js:72`。

```jsx
const emotionsTab = useMemo(() => {
  if (!isChar(selectedType)) return nullTab

  return {
    tab: <Tab><Icon src='icon-tab-emotions'/></Tab>,
    panel: <Panel><EmotionInspector/></Panel>
  }
}, [selectedType])
```

渲染位置在 tabs header/body 中：

- 图标位置：`src/js/shot-generator/components/InspectedElement/index.js:125`
- 内容面板位置：`src/js/shot-generator/components/InspectedElement/index.js:135`

## 数据结构

### 1. 表情预设数据

系统表情定义在 `src/js/shared/reducers/shot-generator-presets/emotions.json`，每个预设包含：

```json
{
  "3287C85D-2DD7-42E5-BCA4-3186B39A713A": {
    "id": "3287C85D-2DD7-42E5-BCA4-3186B39A713A",
    "name": "Angry",
    "keywords": "Angry",
    "priority": 0
  }
}
```

预设元数据只保存 id、名称、关键词和排序权重；真正用于显示/绘制的图片文件通过命名规则找到：

- 系统表情缩略图：`data/shot-generator/emotions/<id>-thumbnail.jpg`
- 系统表情贴图：`data/shot-generator/emotions/<id>-texture.png`
- 用户表情缩略图：用户 preset 目录下 `emotions/<id>-thumbnail.jpg`
- 用户表情贴图：用户 preset 目录下 `emotions/<id>-texture.png`

### 2. 人物对象上的表情字段

选中某个表情后，不会创建新对象，而是更新当前人物 `sceneObject`：

```js
updateObject(sceneObjectId, { emotionPresetId })
```

如果选择“没有表情”，则写入：

```js
updateObject(sceneObjectId, { emotionPresetId: undefined })
```

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:149`。

## 表情列表 UI 流程

`EmotionInspector` 通过 `mapStateToProps` 读取：

- 当前选中人物：`sceneObject`
- 用户自定义表情：`userEmotions`
- 当前人物已选中的预设：`selectedPreset`

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:236`。

```js
const mapStateToProps = state => {
  let sceneObjects = getSceneObjects(state)
  let id = getSelections(state)[0]
  let sceneObject = sceneObjects[id]

  return {
    sceneObject,
    userEmotions: filter(
      ({ id }) => systemEmotions[id] == null,
      state.presets.emotions
    ),
    selectedPreset: state.presets.emotions[sceneObject.emotionPresetId]
  }
}
```

UI 中会把“无表情”、系统表情、用户表情合并排序：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:310`。

```js
const presets = [
  EMOTION_PRESET_NONE
].concat(
  [
    ...Object.values(systemEmotions),
    ...Object.values(userEmotions)
  ]
  .sort(comparePresetNames)
  .sort(comparePresetPriority)
)
```

然后转换成 Grid 使用的元素：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:328`。

```js
const presetsAsGridItems = presets.map(({ id, name, keywords }) => ({
  title: name,
  src: id == null
    ? getAssetPath('emotion', `emotions-none.png`)
    : systemEmotions[id] != null
      ? getAssetPath('emotion', `${id}-thumbnail.jpg`)
      : getUserPresetPath('emotions', `${id}-thumbnail.jpg`),
  isSelected: id == null
    ? selectedPreset == null
    : selectedPreset && selectedPreset.id == id,
  id: id,
  onDelete: id == null || systemEmotions[id] != null ? null : onDeletePreset
}))
```

截图中的 “Angry / Cry / Dead” 等卡片就是这里根据 `emotions.json` 和缩略图路径生成的。

搜索框使用 `SearchList`，点击卡片使用通用 `Grid` + `GridItem`：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:379`。

```jsx
<SearchList
  label={t('shot-generator.inspector.emotions.search-list-label')}
  list={searchList}
  onSearch={onSearch}
/>

<Grid
  itemData={{
    onSelect: ({ id }) => send({
      type: 'SELECT_ITEM',
      sceneObjectId: sceneObject.id,
      emotionPresetId: id
    })
  }}
  Component={GridItem}
  elements={elements}
  numCols={NUM_COLS}
  itemHeight={ITEM_HEIGHT}
/>
```

## 点击表情卡片后发生什么

`EmotionInspector` 使用 XState `machine` 管理 UI 状态。点击 GridItem 后发送事件：

```js
send({
  type: 'SELECT_ITEM',
  sceneObjectId: sceneObject.id,
  emotionPresetId: id
})
```

在 state machine 的 `idle` 状态中，`SELECT_ITEM` 触发 `selectItem` action：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:162`。

```js
idle: {
  on: {
    SELECT_FILE: 'selectFile',
    SELECT_ITEM: {
      actions: selectItem
    },
    DELETE_PRESET: {
      actions: deletePreset
    }
  }
}
```

`selectItem` 会开启 undo 分组，并更新人物 `emotionPresetId`：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:149`。

```js
const selectItem = (context, event) => {
  let { dispatch } = context
  let { sceneObjectId, emotionPresetId } = event

  dispatch(undoGroupStart())
  if (emotionPresetId) {
    dispatch(updateObject(sceneObjectId, { emotionPresetId }))
  } else {
    dispatch(updateObject(sceneObjectId, { emotionPresetId: undefined }))
  }
  dispatch(undoGroupEnd())
}
```

也就是说，表情选择本身只改 Redux 中当前人物的一项字段。

## 渲染时如何把 emotionPresetId 转成图片

`SceneManagerR3fLarge` 渲染人物时读取 `sceneObject.emotionPresetId`，并按系统/用户预设选择贴图路径：

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:383`。

```js
let { emotionPresetId } = sceneObject
let imagePath = emotionPresetId
  ? Object.keys(systemEmotionPresets).includes(emotionPresetId)
    ? getAssetPath('emotion', `${emotionPresetId}-texture.png`)
    : getUserPresetPath('emotions', `${emotionPresetId}-texture.png`)
  : null
```

随后将该路径作为 `imagePath` 传给 `Character`：

```jsx
<Character
  sceneObject={sceneObject}
  imagePath={imagePath}
  ...
/>
```

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:390`。

## Character 如何应用表情纹理

`Character` 内部有一个 `FaceMesh` 实例，用于处理脸部贴图合成：

参考：`src/js/shot-generator/components/Three/Character.js:43`。

```js
const faceMesh = useRef(null)
function getFaceMesh () {
  if (faceMesh.current === null) {
    faceMesh.current = new FaceMesh()
  }
  return faceMesh.current
}
```

当人物 glTF 加载完成并生成 `LOD` 时，内置人物会初始化 `FaceMesh`：

参考：`src/js/shot-generator/components/Three/Character.js:75`。

```js
if (isUserModel(sceneObject.model)) {
  originalHeight = 1
} else {
  getFaceMesh().setSkinnedMesh(lod, gl)
  let bbox = new THREE.Box3().setFromObject(lod)
  originalHeight = bbox.max.y - bbox.min.y
}
```

注意：这里自定义人物模型不会走 `setSkinnedMesh(lod)` 的分支，因此这套表情贴图方案主要面向内置人物模型。

`Character` 使用 `useAsset` 加载 `imagePath`：

参考：`src/js/shot-generator/components/Three/Character.js:60`。

```js
const {asset: texture} = useAsset(ready ? props.imagePath : null)
```

当 `texture` 或 `lod` 变化时：

- 没有表情贴图：调用 `resetTexture()` 恢复原始皮肤贴图。
- 有表情贴图：调用 `draw(texture)` 把表情图片绘制到脸上。

参考：`src/js/shot-generator/components/Three/Character.js:431`。

```js
useEffect(() => {
  if(!skeleton) return
  if(!texture) {
    getFaceMesh().resetTexture()
    return
  }
  getFaceMesh().draw(texture)
}, [texture, lod])
```

## FaceMesh 的核心算法

`FaceMesh` 是整个表情功能的底层实现，路径：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js`。

### 1. 初始化：把人物原始贴图复制到 canvas

`setSkinnedMesh(lod)`：

1. 取 `lod.children[0]` 作为用于定位脸部的 `SkinnedMesh`。
2. 保存原始材质贴图为 `defaultTexture`。
3. 创建一个和原图同尺寸的 `canvas`。
4. 创建 `new THREE.Texture(this.drawingCanvas)`。
5. 把 LOD 下所有 mesh 的 `material.map` 替换成这个 canvas texture。

参考：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js:98`。

```js
setSkinnedMesh(lod) {
  this.lod = lod
  this.skinnedMesh = lod.children[0]
  this.image = this.skinnedMesh.material.map.image

  this.defaultTexture = this.skinnedMesh.material.map.clone()
  this.drawingCanvas.width = this.image.width
  this.drawingCanvas.height = this.image.height
  let texture = new THREE.Texture(this.drawingCanvas)
  texture.flipY = false
  this.drawingCtx.drawImage(this.image, 0, 0, this.image.width, this.image.height)

  for(let i = 0; i < this.lod.children.length; i++) {
    this.lod.children[i].material.map = texture
    this.lod.children[i].material.map.needsUpdate = true
  }
}
```

这一步让后续所有表情绘制都可以直接改 canvas，Three.js 材质会显示更新后的 canvas texture。

### 2. 找脸部中心对应的 UV

`draw(texture)` 中先从骨架取出：

- `Head`
- `LeftEye`
- `RightEye`

然后取三者的中点作为脸部大致中心：

参考：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js:216`。

```js
let headBone = this.skinnedMesh.skeleton.getBoneByName('Head')
let leftEye = this.skinnedMesh.skeleton.getBoneByName('LeftEye')
let rightEye = this.skinnedMesh.skeleton.getBoneByName('RightEye')
let rightEyePosition = rightEye.worldPosition()
let leftEyePosition = leftEye.worldPosition()
let headPosition = headBone.worldPosition()
let position = getMidpoint(headPosition, leftEyePosition, rightEyePosition)
let intersect = this.facesSearch(position, headBone)
```

`facesSearch(position, headBone)` 会从头部前方沿头部方向反向投射一条 ray，遍历 skinned mesh 的三角面，找到命中的脸部三角形，并计算该命中点的 UV。

参考：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js:129`。

这个过程会考虑：

- 几何 position attribute
- skinning 变换：`object.boneTransform(...)`
- morph target 变形：`applyMorph(...)`
- uv / uv2
- mesh world matrix

相关代码：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js:153`。

```js
_vA.fromBufferAttribute(position, a)
_vB.fromBufferAttribute(position, b)
_vC.fromBufferAttribute(position, c)

var morphInfluences = object.morphTargetInfluences
if (morphInfluences && morphPosition) {
  applyMorph(_vA, _vB, _vC, morphPosition, morphInfluences)
}

object.boneTransform(a, _vA)
object.boneTransform(b, _vB)
object.boneTransform(c, _vC)

intersect = ray.intersectTriangle(_vA, _vB, _vC, ...)
intersect.uv = THREE.Triangle.getUV(target, _vA, _vB, _vC, _uvA, _uvB, _uvC, new THREE.Vector2())
```

### 3. 把表情图画到原始脸部贴图上

拿到 UV 后，`draw(texture)` 将 UV 转换为原始贴图像素坐标：

```js
let meshPos = {
  x: uv.x * this.image.width,
  y: uv.y * this.image.height
}
```

然后将表情图片缩放到最大 512px，并画到 canvas 上：

参考：`src/js/shot-generator/components/Three/Helpers/FaceMesh.js:232`。

```js
this.drawingCtx.drawImage(this.image, 0, 0, this.image.width, this.image.height)
this.drawingCtx.drawImage(
  emotionImage,
  meshPos.x - width / 2,
  meshPos.y - height / 2,
  width,
  height
)
```

最后标记贴图需要更新：

```js
for(let i = 0; i < this.lod.children.length; i++) {
  this.lod.children[i].material.map.needsUpdate = true
}
```

## 用户自定义表情导入流程

截图里搜索框旁边的 `+` 按钮用于导入自定义表情图片。

点击 `+` 时发送 `SELECT_FILE`：

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:387`。

```jsx
<a className="button_add" href="#"
  onPointerDown={() => send({
    type: 'SELECT_FILE',
    sceneObjectId: sceneObject.id
  })}
>+</a>
```

XState 流程：

1. `selectFile`：调用 Electron `remote.dialog.showOpenDialog` 选择图片文件。
2. `prompt`：弹窗让用户填写 preset 名称。
3. `save`：调用 `createPreset`。

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:162`。

`createPreset` 做了几件事：

1. 确保用户 presets/emotions 目录存在。
2. 生成 uuid 作为表情 preset id。
3. 将源图片复制为 `<id>-texture.png`。
4. 加载该图片为 `THREE.Texture`。
5. 使用 `EmotionPresetThumbnailRenderer` 生成 `<id>-thumbnail.jpg`。
6. `dispatch(createEmotionPreset(emotionPreset))` 写入 Redux preset 数据。
7. `dispatch(writeEmotionPresets())` 将用户 preset 元数据保存到本地 JSON。
8. 自动选择这个新表情，更新当前人物 `emotionPresetId`。

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:81`。

```js
let emotionPreset = emotionPresetFactory({ name })
let dest = getUserPresetPath('emotions', `${emotionPreset.id}-texture.png`)
fs.copyFileSync(src, dest)

let faceTexture = await new Promise((resolve, reject) => {
  new THREE.TextureLoader().load(dest, resolve, null, reject)
})

let thumbnailFilePath = getUserPresetPath('emotions', `${emotionPreset.id}-thumbnail.jpg`)
getThumbnailRenderer().render({ faceTexture })
fs.writeFileSync(
  thumbnailFilePath,
  getThumbnailRenderer().toBase64('image/jpg'),
  'base64'
)

dispatch(createEmotionPreset(emotionPreset))
dispatch(writeEmotionPresets())
```

## 缩略图生成机制

`EmotionPresetThumbnailRenderer` 会创建一个默认人物模型，把表情图用同一个 `FaceMesh.draw(faceTexture)` 画到脸上，然后用 `ThumbnailRenderer` 渲染成 jpg。

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/thumbnail-renderer.js:78`。

```js
class EmotionPresetThumbnailRenderer {
  constructor ({ characterGltf, inverseSide }) {
    this.thumbnailRenderer = new ThumbnailRenderer({ inverseSide })

    let characterGroup = createCharacter(characterGltf)
    this.thumbnailRenderer.getGroup().add(characterGroup)

    let character = characterGroup.getObjectByProperty('type', 'SkinnedMesh')

    this.faceMesh = new FaceMesh()
    this.faceMesh.setSkinnedMesh({ children: [character] })
  }

  render ({ faceTexture }) {
    this.faceMesh.draw(faceTexture)
    this.thumbnailRenderer.render()
  }
}
```

系统表情缩略图也有类似生成逻辑，测试/工具入口可参考：`test/views/thumbnail-renderer/window.js`。

## 删除自定义表情

系统预设不可删除；用户预设的 GridItem 会带 `onDelete`。

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:328`。

```js
onDelete: id == null || systemEmotions[id] != null ? null : onDeletePreset
```

删除时：

1. 弹确认框。
2. dispatch `DELETE_EMOTION_PRESET`。
3. 保存用户 presets JSON。
4. 删除 `<id>-texture.png` 和 `<id>-thumbnail.jpg`。

参考：`src/js/shot-generator/components/InspectedElement/EmotionInspector/index.js:131`。

Reducer 里还有清理引用逻辑：当某个情绪 preset 被删除时，所有正在引用它的 scene object 都会删除 `emotionPresetId`。

参考：`src/js/shared/reducers/shot-generator.js:1213`。

```js
case 'DELETE_EMOTION_PRESET':
  for (let id in state) {
    if (state[id].emotionPresetId === action.payload.id) {
      delete draft[id].emotionPresetId
    }
  }
  return
```

Preset 数据本身由 presets reducer 管理：

参考：`src/js/shared/reducers/shot-generator.js:1441`。

```js
case 'CREATE_EMOTION_PRESET':
  draft.emotions[action.payload.id] = action.payload
  return
case 'DELETE_EMOTION_PRESET':
  delete draft.emotions[action.payload.id]
  return
```

## 和 morphTargets 的关系

代码中人物材质启用了 `morphTargets`，但截图中这个情绪/表情 Tab 不是通过 `morphTargets` 切换表情。`morphTargets` 主要用于人物体型 morph：

参考：`src/js/shot-generator/components/Three/Character.js:318`。

```js
modelSettings.validMorphTargets.forEach((name, index) => {
  skinnedMesh.morphTargetInfluences[index] = sceneObject.morphTargets[name]
})
```

`FaceMesh.facesSearch` 在寻找脸部 UV 时会考虑当前 morph 结果，目的是让“贴图定位”跟随变形后的 mesh，而不是用 morph target 表达表情。

## 可复用参考代码

如果要复刻“选择表情并应用到人物”的数据层流程，可以参考：

```js
function selectEmotionPreset(dispatch, sceneObjectId, emotionPresetId) {
  dispatch(undoGroupStart())
  dispatch(updateObject(sceneObjectId, {
    emotionPresetId: emotionPresetId || undefined
  }))
  dispatch(undoGroupEnd())
}
```

如果要根据人物对象解析表情贴图路径，可以参考：

```js
function getEmotionTexturePath({ emotionPresetId, systemEmotionPresets, getAssetPath, getUserPresetPath }) {
  if (!emotionPresetId) return null

  return Object.keys(systemEmotionPresets).includes(emotionPresetId)
    ? getAssetPath('emotion', `${emotionPresetId}-texture.png`)
    : getUserPresetPath('emotions', `${emotionPresetId}-texture.png`)
}
```

如果要把表情图合成到人物材质贴图上，核心思路是：

```js
const faceMesh = new FaceMesh()
faceMesh.setSkinnedMesh(lod)

// texture 是表情图片的 THREE.Texture
faceMesh.draw(texture)

// 清除表情时恢复原始贴图
faceMesh.resetTexture()
```

## 注意点

- 这套功能依赖人物骨骼中存在 `Head`、`LeftEye`、`RightEye`，否则 `FaceMesh.draw` 无法定位脸部中心。
- 表情图必须是图片纹理，不是姿态数据；系统默认文件命名必须符合 `<id>-texture.png` 和 `<id>-thumbnail.jpg`。
- 自定义表情 preset 元数据写到用户 presets store，图片文件写到用户 presets/emotions 目录。
- 自定义人物模型目前不会像内置人物那样初始化 `FaceMesh.setSkinnedMesh(lod)`，因此情绪贴图功能主要适配内置人物。
- 表情绘制会覆盖人物材质 `map` 为 canvas texture；每次切换表情都会先重绘原始皮肤图，再叠加表情图片。
