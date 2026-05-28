# Shot Generator 头发 Tab 技术实现分析

本文分析 Shot Generator 人物编辑状态下，截图中“胡子/头发”图标的 Hair Tab 如何工作，包括：头发预设列表、选择/取消头发、自定义头发模型导入，以及头发如何绑定到人物头部。

## 结论

头发 Tab 的实现复用了 Shot Generator 的 `attachable` 体系。它不是人物 mesh 的材质切换，也不是 morph target，而是一个特殊类型的可附着 3D 模型：

1. 头发模型在数据中是 `type: 'attachable'`，并额外标记 `attachableType: 'hair'`。
2. 选择头发时，会给当前人物创建一个新的 `sceneObject`，类型仍然是 `attachable`。
3. 这个头发对象固定 `bindBone: 'Head'`，并用 `attachToId` 指向当前人物 id。
4. 场景渲染时仍由通用 `Attachable` 组件负责加载模型、找到人物骨骼，并执行 `bone.add(ref.current)` 绑定到头部骨骼。
5. 一个角色同一时间只保留一个头发对象：选择新头发会先删除旧头发；选择“无头发”会删除当前头发。
6. 自定义头发会被复制到项目的 `models/attachables` 目录，再作为 `attachable` 模型创建。

所以，Hair Tab 可以理解为一个针对 `attachableType: 'hair'` 做了特殊 UI 和替换逻辑的配件 Tab。

## 关键代码位置

- Tab 注册入口：`src/js/shot-generator/components/InspectedElement/index.js`
- 头发列表、搜索、选择、导入：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js`
- 头发模型预设数据：`src/data/shot-generator/attachables/attachables.json`
- 通用 attachable 渲染与骨骼绑定：`src/js/shot-generator/components/Three/Attachable.js`
- 场景中挂载 attachable：`src/js/shot-generator/SceneManagerR3fLarge.js`
- 文件复制工具：`src/js/shot-generator/utils/CopyFile.js`
- 路径解析：`src/js/shot-generator/services/filepaths.js`
- Redux actions/reducers：`src/js/shared/reducers/shot-generator.js`

## Tab 是如何挂到人物属性面板的

人物属性面板在 `InspectedElement/index.js` 中注册 Hair Tab。只有选中对象是人物时才显示：

参考：`src/js/shot-generator/components/InspectedElement/index.js:81`。

```jsx
const hairInspectorTab = useMemo(() => {
  if (!isChar(selectedType)) return nullTab

  return {
    tab: <Tab><Icon src='icon-tab-hair'/></Tab>,
    panel: <Panel><HairInspector /></Panel>
  }
}, [selectedType])
```

最终渲染到：

- Tab 图标区域：`src/js/shot-generator/components/InspectedElement/index.js:126`
- Tab 内容区域：`src/js/shot-generator/components/InspectedElement/index.js:136`

## 数据结构

### 1. 头发预设数据

系统头发模型定义在 `src/data/shot-generator/attachables/attachables.json`。示例：

参考：`src/data/shot-generator/attachables/attachables.json:53`。

```json
"hair": {
  "id": "hair",
  "name": "Hair: Default",
  "type": "attachable",
  "x": -0.0013,
  "y": 0.15,
  "z": 0.014,
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "bindBone": "Head",
  "attachableType": "hair"
}
```

关键字段：

- `type: 'attachable'`：仍然是通用可附着对象。
- `attachableType: 'hair'`：标记这是头发，Hair Tab 只筛选这类模型。
- `bindBone: 'Head'`：默认绑定到头部骨骼。
- `x/y/z`、`rotation`：相对于头部绑定逻辑的初始摆放数据，之后会被转换成世界位置/旋转保存。

### 2. 创建到场景中的头发对象

`HairInspector` 中的 `createSceneObjectForAttachable()` 提供头发对象的默认结构：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:45`。

```js
const createSceneObjectForAttachable = () => {
  return {
    id: THREE.Math.generateUUID(),

    type: 'attachable',
    attachableType: 'hair',
    bindBone: 'Head',

    x: 0,
    y: 0,
    z: 0,
    rotation: { x: 0, y: 0, z: 0 },

    size: 1,
    status: 'PENDING'
  }
}
```

选择具体头发时，会再补充：

- `model`：头发模型 id 或自定义模型路径。
- `attachToId`：当前人物 id。
- `sceneObjectOverrides`：预设里带来的 `name/x/y/z/rotation` 或自定义头发的位置修正。

## HairInspector 如何读取当前人物和头发

`HairInspector` 通过 Redux connect 读取四类数据：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:87`。

```js
(state) => ({
  selectedSceneObject: getFirstSelectedSceneObject(state),
  selectedHair: getSelectedHair(state),
  attachableHairModels: getAttachableHairModels(state),
  storyboarderFilePath: state.meta.storyboarderFilePath
})
```

### 1. 当前人物

`getFirstSelectedSceneObject` 使用当前 selection 的第一个 id 查找人物对象：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:69`。

```js
const getFirstSelectedSceneObject = createSelector(
  [getSceneObjects, getSelections],
  (sceneObjects, selections) => sceneObjects[selections[0]]
)
```

### 2. 当前人物已绑定的头发

`getSelectedHair` 在全部 `sceneObjects` 中查找：

- `type === 'attachable'`
- `attachableType === 'hair'`
- `attachToId === 当前选中人物 id`

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:74`。

```js
const getSelectedHair = createSelector(
  [getSceneObjects, getSelections],
  (sceneObjects, selections) =>
    selections[0]
      ? Object.values(sceneObjects).find(
          (item) =>
            item.type === 'attachable' &&
            item.attachableType === 'hair' &&
            item.attachToId === selections[0]
        )
      : undefined
)
```

这也是为什么一个人物只显示一个当前头发：这里使用的是 `find`，选择新头发时也会删除旧头发。

### 3. 可选头发模型列表

Hair Tab 从全局 `state.models` 里筛选所有 attachable，再只保留 `attachableType === 'hair'`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:63`。

```js
const getAttachableModels = (state) =>
  Object.values(state.models).filter((m) => m.type === 'attachable')

const getAttachableHairModels = (state) =>
  getAttachableModels(state).filter((m) => m.attachableType === 'hair')
```

## UI 列表如何生成

Hair Tab 会构造一个 `modelsList`，第一项是“无头发”，后面拼接系统头发模型：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:181`。

```js
let modelsList = [
  {
    id: null,
    model: null,
    keywords: '',
    name: t('shot-generator.inspector.hair.no-value')
  }
].concat(attachableHairModels)
```

搜索框使用 `SearchList`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:190`。

```js
let searchList = useMemo(
  () => modelsList.map(({ id, name, keywords }) => ({
    value: [name, keywords].filter(Boolean).join(' '),
    id
  })),
  attachableHairModels
)
```

Grid 元素会把 `Hair: Default` 这种名称显示为 `Default`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:205`。

```js
elements = matches.map((attachable) => ({
  title: attachable.name.replace(/^Hair:\s+/, ''),
  src: attachable.id == null
    ? GRID_ITEM_NONE_SRC
    : getAssetPath(attachable.type, `${attachable.id}.jpg`),
  isSelected: attachable.id == null
    ? selectedHair == null
    : selectedHair && attachable.id === selectedHair.model,
  model: attachable.id,
  sceneObjectOverrides: {
    name: attachable.name,
    x: attachable.x,
    y: attachable.y,
    z: attachable.z,
    rotation: attachable.rotation
  }
}))
```

缩略图路径使用 attachable 资源目录：

- 无头发：`getAssetPath('attachable', 'hair-none.png')`
- 系统头发：`getAssetPath('attachable', '<id>.jpg')`

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:120`、`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:208`。

最终用通用 Grid 渲染：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:281`。

```jsx
<Grid
  itemData={{ onSelect }}
  Component={GridItem}
  elements={elements}
  numCols={itemSettings.NUM_COLS}
  itemHeight={itemSettings.ITEM_HEIGHT}
/>
```

## 点击头发卡片后发生什么

核心逻辑在 `onSelect(data)`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:122`。

### 1. 选择某个头发

当 `data && data.model` 存在时，表示用户选择了具体头发：

```js
if (data && data.model) {
  undoGroupStart()

  if (selectedHair) {
    deleteObjects([selectedHair.id])
    deselectAttachable()
  }

  let sceneObject = {
    ...createSceneObjectForAttachable(),
    model: data.model,
    attachToId: selectedSceneObject.id,
    ...data.sceneObjectOverrides
  }

  createObject(sceneObject)
  selectAttachable({
    id: sceneObject.id,
    bindId: sceneObject.attachToId
  })

  undoGroupEnd()
}
```

具体行为：

1. 开启 undo 分组。
2. 如果当前人物已有头发，先删除旧头发并取消 attachable 选中。
3. 创建新的 `type: 'attachable'`、`attachableType: 'hair'` 对象。
4. 设置 `attachToId` 为当前人物 id。
5. 调用 `createObject(sceneObject)` 写入 Redux。
6. 调用 `selectAttachable({ id, bindId })` 选中新头发，同时保持人物为当前 selection。
7. 结束 undo 分组。

### 2. 选择“无头发”

当第一项“无头发”被点击时，`data.model` 为空，走 else 分支：

```js
if (selectedHair) {
  undoGroupStart()
  deleteObjects([selectedHair.id])
  deselectAttachable()
  undoGroupEnd()
}
```

也就是只删除当前人物的头发 attachable，不创建新对象。

## 自定义头发导入流程

截图里右侧“选择文件...”使用 `FileInput`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:256`。

```jsx
<FileInput
  value={
    isCustom
      ? shortBaseName(selectedHair.model)
      : t('shot-generator.inspector.common.select-file')
  }
  onChange={onSelectFile}
  refClassName={refClassName}
  wrapperClassName={wrapperClassName}
/>
```

是否自定义模型通过 `isUserModel(selectedHair.model)` 判断：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:227`。

```js
let isCustom = selectedHair && isUserModel(selectedHair.model)
```

选择文件后执行 `onSelectFile(filepath)`：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:158`。

```js
const onSelectFile = useCallback(
  (filepath) => {
    if (filepath.file) {
      let model = CopyFile(
        storyboarderFilePath,
        filepath.file,
        'attachable'
      )
      onSelect({
        model,
        sceneObjectOverrides: {
          name: path.basename(model, path.extname(model)),
          ...USER_MODEL_HAIR_POSITION
        }
      })
    }
  },
  [storyboarderFilePath, onSelect]
)
```

`CopyFile` 会在需要时把源模型复制到项目目录：

参考：`src/js/shot-generator/utils/CopyFile.js:7`。

```js
let dst = path.join(
  path.dirname(storyboarderFilePath),
  ModelLoader.projectFolder(type),
  path.basename(expectedFilepath)
)
fs.ensureDirSync(path.dirname(dst))
fs.copySync(src, dst, { overwrite: true, errorOnExist: false })
return path.join(
  ModelLoader.projectFolder(type),
  path.basename(dst)
).split('\\').join('/')
```

因为这里传入的 `type` 是 `'attachable'`，所以自定义头发模型会进入 attachable 模型目录，而不是单独的 hair 目录。

自定义头发使用固定初始位置：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:36`。

```js
const USER_MODEL_HAIR_POSITION = {
  x: -0.0013,
  y: 0.15,
  z: 0.014
}
```

## 头发如何绑定到人物头上

Hair Tab 自己不做 Three.js 绑定，它只创建一个特殊的 attachable 数据对象。真正绑定发生在通用 `Attachable` 组件中。

### 1. SceneManager 渲染 attachable

`SceneManagerR3fLarge` 会渲染所有 `type === 'attachable'` 的对象，头发也包含在内：

参考：`src/js/shot-generator/SceneManagerR3fLarge.js:423`。

```jsx
<Attachable
  path={ModelLoader.getFilepathForModel(sceneObject, {storyboarderFilePath})}
  sceneObject={sceneObject}
  isSelected={selectedAttachable === sceneObject.id}
  updateObject={updateObject}
  сharacterModelPath={ModelLoader.getFilepathForModel(sceneObjects[sceneObject.attachToId], {storyboarderFilePath})}
  deleteObjects={deleteObjects}
  character={sceneObjects[sceneObject.attachToId]}
  withState={withState}
/>
```

### 2. Attachable 绑定 Head 骨骼

由于头发对象有：

```js
attachableType: 'hair',
bindBone: 'Head'
```

`Attachable` 初始化时会找到目标人物的 `SkinnedMesh`，再找 `Head` 骨骼，最后执行：

参考：`src/js/shot-generator/components/Three/Attachable.js:140`。

```js
let skinnedMesh = characterObject.current.getObjectByProperty('type', 'SkinnedMesh')
let bone = skinnedMesh.skeleton.bones.find(b => b.name === sceneObject.bindBone)

// 首次 PENDING 时计算初始位置/旋转，并 updateObject(..., { status: 'Loaded' })

bone.add(ref.current)
```

这就是头发跟随头部运动的核心。人物切换姿势、转头或缩放时，头发作为 `Head` 骨骼的子节点会继承骨骼矩阵。

### 3. 头发有一条特殊初始化分支

`Attachable` 对“用户自定义非头发模型”和“头发模型”处理不同：

参考：`src/js/shot-generator/components/Three/Attachable.js:149`。

```js
if (
  isUserModel(sceneObject.model) &&
  !(sceneObject.attachableType === 'hair' && sceneObject.bindBone === 'Head')
) {
  modelPosition.copy(bone.worldPosition())
  quat = bone.worldQuaternion()
} else {
  // built-in system models, or custom models with hair
  let {x, y, z} = sceneObject
  modelPosition.set(x, y, z)
  // 按预设 x/y/z/rotation 和人物缩放计算最终世界位置
}
```

也就是说：

- 普通自定义 attachable 默认放到骨骼世界位置。
- 头发即使是自定义模型，只要 `attachableType === 'hair' && bindBone === 'Head'`，也会走预设偏移逻辑，使用 `USER_MODEL_HAIR_POSITION` 或系统预设里的 `x/y/z`。

这保证了自定义头发导入后会出现在头顶附近，而不是直接落在 Head 骨骼原点。

## 和普通配件 Tab 的关系

普通配件 Tab 在 `AttachableInspector` 里主动排除了头发：

参考：`src/js/shot-generator/components/InspectedElement/AttachableInspector/index.js:59`。

```js
Object.values(allModels).filter(m => m.type === 'attachable' && m.attachableType !== 'hair')
```

头发 Tab 则只保留头发：

参考：`src/js/shot-generator/components/InspectedElement/HairInspector/index.js:66`。

```js
getAttachableModels(state).filter((m) => m.attachableType === 'hair')
```

普通配件编辑器也不会列出头发：

参考：`src/js/shot-generator/components/InspectedElement/AttachableEditor/index.js:75`。

```js
if(value.attachToId === sceneObject.id && value.attachableType !== 'hair')
  result.push(value)
```

因此 UI 上头发和普通配件是两个入口，但底层都走 attachable 渲染和骨骼绑定。

## 路径规则

`getAssetPath('attachable', ...)` 的系统资源目录是：

参考：`src/js/shot-generator/services/filepaths.js:3`。

```js
'attachable': path.join(pathToShotGeneratorData, 'attachables')
```

项目内自定义 attachable 目录是：

参考：`src/js/shot-generator/services/filepaths.js:16`。

```js
'attachable': path.join(base, 'models', 'attachables')
```

`createAssetPathResolver` 会根据 filepath 中是否包含 `/` 判断系统资源还是用户/项目资源：

参考：`src/js/shot-generator/services/filepaths.js:60`。

```js
if (filepath.match(/\//)) {
  return path.join(projectDirectoryFor(type), path.basename(filepath))
} else {
  return path.join(systemDirectoryFor(type), path.basename(filepath))
}
```

所以：

- 系统头发 `hair` 的模型/缩略图来自 `data/shot-generator/attachables`。
- 自定义头发模型路径会类似 `models/attachables/foo.glb`，最终解析到项目目录下。

## 删除联动

删除人物时，Redux reducer 会删除绑定在该人物上的所有 attachable，头发也会被一起删掉：

参考：`src/js/shared/reducers/shot-generator.js:974`。

```js
if (draft[id].type === 'character') {
  let attachableIds = Object.values(draft)
    .filter(obj => obj.attachToId === id)
    .map(obj => obj.id)

  for (let attachableId of attachableIds) {
    delete draft[attachableId]
  }
}
```

## 可复用参考代码

如果要创建一个绑定到头部的头发对象，可以参考：

```js
function createHairSceneObject({ characterId, model, overrides = {} }) {
  return {
    id: THREE.Math.generateUUID(),
    type: 'attachable',
    attachableType: 'hair',
    bindBone: 'Head',
    x: 0,
    y: 0,
    z: 0,
    rotation: { x: 0, y: 0, z: 0 },
    size: 1,
    status: 'PENDING',
    model,
    attachToId: characterId,
    ...overrides
  }
}
```

如果要复刻 Hair Tab 的“单头发替换”逻辑，可以参考：

```js
function replaceCharacterHair({ dispatch, selectedHair, characterId, model, overrides }) {
  dispatch(undoGroupStart())

  if (selectedHair) {
    dispatch(deleteObjects([selectedHair.id]))
    dispatch(deselectAttachable())
  }

  const hair = createHairSceneObject({ characterId, model, overrides })

  dispatch(createObject(hair))
  dispatch(selectAttachable({ id: hair.id, bindId: characterId }))

  dispatch(undoGroupEnd())
}
```

如果要取消头发：

```js
function removeCharacterHair({ dispatch, selectedHair }) {
  if (!selectedHair) return

  dispatch(undoGroupStart())
  dispatch(deleteObjects([selectedHair.id]))
  dispatch(deselectAttachable())
  dispatch(undoGroupEnd())
}
```

## 注意点

- 头发和普通配件底层都是 `attachable`，区别主要是 `attachableType: 'hair'` 和 `bindBone: 'Head'`。
- 当前 Hair Tab 的逻辑默认一个人物只有一个头发；选择新头发会删除旧头发。
- 头发不会出现在普通 AttachableEditor 列表里，避免用户在配件页修改头发绑定骨骼。
- 自定义头发也是 attachable 模型，会复制到项目 `models/attachables` 目录。
- 自定义头发使用 `USER_MODEL_HAIR_POSITION` 作为默认偏移；如果模型自身尺度或原点不符合预期，导入后可能需要模型文件本身调整。
- 头发依赖目标人物骨架中存在 `Head` 骨骼；如果自定义人物缺少该骨骼，通用 `Attachable` 绑定会失败或无法正确定位。
