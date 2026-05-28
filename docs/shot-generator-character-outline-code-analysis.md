# Shot Generator：人物模型黑边 / Outline 代码实现分析

## 1. 结论概览

截图中人物模型边缘的黑色描边不是模型贴图自带的线条，也不是后期图像边缘检测，而是 Three.js 的 `OutlineEffect` 风格渲染实现。

当前实现可以概括为：

```text
人物模型加载
  -> 替换为 MeshToonMaterial
  -> patchMaterial(material) 写入 material.userData.outlineParameters
  -> 主视图使用 useShadingEffect 选择 OutlineEffect
  -> OutlineEffect 先正常 render 一遍原模型
  -> 再遍历 scene，把带 outlineParameters 的材质临时替换成 BackSide 描边 ShaderMaterial
  -> 第二遍 render 只画向外扩张的背面轮廓
  -> restoreOriginalMaterial 还原原材质
```

黑边的关键条件有两个：

1. 材质必须通过 `patchMaterial` 写入 `material.userData.outlineParameters`。
2. 当前渲染器必须是 `OutlineEffect`，也就是 `world.shadingMode === ShadingType.Outline` 或特定视图强制使用 Outline。

## 2. 核心文件与代码位置

| 文件 | 代码位置 | 作用 |
| --- | --- | --- |
| `src/js/shot-generator/components/Three/Character.js` | `src/js/shot-generator/components/Three/Character.js:80` 到 `src/js/shot-generator/components/Three/Character.js:126` | 加载 / clone 角色 GLTF，筛选 SkinnedMesh，替换为 `MeshToonMaterial`，并调用 `patchMaterial` 开启描边参数。 |
| `src/js/shot-generator/components/Three/Character.js` | `src/js/shot-generator/components/Three/Character.js:391` 到 `src/js/shot-generator/components/Three/Character.js:394` | 角色选中状态变化时调用 `setSelected`，改变描边颜色和材质颜色。 |
| `src/js/shot-generator/helpers/outlineMaterial.js` | `src/js/shot-generator/helpers/outlineMaterial.js:2` 到 `src/js/shot-generator/helpers/outlineMaterial.js:12` | 定义默认黑色描边、选中描边色，并通过 `patchMaterial` 写入 `outlineParameters`。 |
| `src/js/shot-generator/helpers/outlineMaterial.js` | `src/js/shot-generator/helpers/outlineMaterial.js:15` 到 `src/js/shot-generator/helpers/outlineMaterial.js:29` | `setSelected` 根据选中状态更新 `outlineParameters.color`，并更新基础材质颜色。 |
| `src/js/vendor/shading-effects/ShadingType.js` | `src/js/vendor/shading-effects/ShadingType.js:1` 到 `src/js/vendor/shading-effects/ShadingType.js:6` | 定义 `Outline` / `Wireframe` / `Flat` / `Depth` 四种渲染模式。 |
| `src/js/vendor/shading-effects/createShadingEffect.js` | `src/js/vendor/shading-effects/createShadingEffect.js:7` 到 `src/js/vendor/shading-effects/createShadingEffect.js:24` | 根据 `shadingMode` 创建对应渲染器；`Outline` 模式创建 `new OutlineEffect(gl, { defaultThickness: 0.015 })`。 |
| `src/js/shot-generator/hooks/use-shading-effect.js` | `src/js/shot-generator/hooks/use-shading-effect.js:6` 到 `src/js/shot-generator/hooks/use-shading-effect.js:11` | 为同一个 WebGLRenderer 预创建四种 shading effect 实例。 |
| `src/js/shot-generator/hooks/use-shading-effect.js` | `src/js/shot-generator/hooks/use-shading-effect.js:28` 到 `src/js/shot-generator/hooks/use-shading-effect.js:50` | 根据当前 `shadingMode` 返回正在使用的 effect 引用。 |
| `src/js/shot-generator/SceneManagerR3fLarge.js` | `src/js/shot-generator/SceneManagerR3fLarge.js:323` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:338` | 主 3D 视图选择 shading effect，并在每帧调用 `renderer.current.render(scene, camera)`。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:64` 到 `src/js/vendor/OutlineEffect.js:109` | `OutlineEffect` 初始化默认描边参数、缓存和 shader uniform。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:112` 到 `src/js/vendor/OutlineEffect.js:155` | 注入顶点 shader 逻辑：沿法线方向在裁剪空间扩张顶点。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:220` 到 `src/js/vendor/OutlineEffect.js:247` | 创建描边 `ShaderMaterial`，使用 `THREE.BackSide` 渲染外壳。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:251` 到 `src/js/vendor/OutlineEffect.js:319` | 按原材质 uuid 缓存 / 获取描边材质，并临时替换 mesh material。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:356` 到 `src/js/vendor/OutlineEffect.js:408` | 从原材质的 `outlineParameters` 同步 `thickness`、`color`、`alpha`、可见性等参数。 |
| `src/js/vendor/OutlineEffect.js` | `src/js/vendor/OutlineEffect.js:460` 到 `src/js/vendor/OutlineEffect.js:525` | 两遍渲染的核心：先正常渲染，再 `renderOutline` 渲染描边，最后还原 scene 状态。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:594` | 默认场景 `world.shadingMode` 是 `ShadingType.Outline`。 |
| `src/js/shared/reducers/shot-generator.js` | `src/js/shared/reducers/shot-generator.js:1372` 到 `src/js/shared/reducers/shot-generator.js:1389` | `CYCLE_SHADING_MODE` 在 Outline / Wireframe / Flat / Depth 之间切换。 |
| `src/js/shot-generator/components/Three/ModelObject.js` | `src/js/shot-generator/components/Three/ModelObject.js:17` 到 `src/js/shot-generator/components/Three/ModelObject.js:25` | 物体模型也使用同一套 `patchMaterial` 描边机制。 |
| `src/js/shot-generator/components/Three/Attachable.js` | `src/js/shot-generator/components/Three/Attachable.js:11` 到 `src/js/shot-generator/components/Three/Attachable.js:20` | 可附着道具也使用同一套 `patchMaterial` 描边机制。 |

## 3. 人物模型材质如何开启黑边

文件：`src/js/shot-generator/components/Three/Character.js`

角色组件在加载 GLTF 后会 clone 模型，并筛选 `SkinnedMesh`：

- `src/js/shot-generator/components/Three/Character.js:80` 到 `src/js/shot-generator/components/Three/Character.js:96`

对于内置角色，`SkinnedMesh` 通常是 GLTF scene 的直接子级；对于自定义角色，如果直接子级找不到 skinned mesh，会通过 `scene.traverse` 继续向下查找。

随后，代码为每个参与渲染的 mesh 替换材质：

- `src/js/shot-generator/components/Three/Character.js:105` 到 `src/js/shot-generator/components/Three/Character.js:123`

```js
mesh.material = new THREE.MeshToonMaterial({
  map: map,
  color: 0xffffff,
  emissive: 0x0,
  specular: 0x0,
  skinning: true,
  shininess: 0,
  flatShading: false,
  morphNormals: true,
  morphTargets: true
})

patchMaterial(mesh.material)
```

这里有两个关键点：

1. 人物使用 `MeshToonMaterial`，所以基础渲染就是偏卡通 / 平涂风格。
2. `patchMaterial(mesh.material)` 给材质写入描边参数，这是后续 `OutlineEffect` 判断“是否需要画轮廓”的标记。

## 4. patchMaterial 与 setSelected

文件：`src/js/shot-generator/helpers/outlineMaterial.js`

### 4.1 patchMaterial

关键位置：

- `src/js/shot-generator/helpers/outlineMaterial.js:2` 到 `src/js/shot-generator/helpers/outlineMaterial.js:12`

```js
const SELECTED_COLOR = [122/256.0/2, 114/256.0/2, 233/256.0/2]
const DEFAULT_COLOR = [0.0, 0.0, 0.0]

export const patchMaterial = (material, customParameters = {}) => {
  material.userData.outlineParameters = {
    thickness: 0.008,
    color: DEFAULT_COLOR,
    ...customParameters
  }

  return material
}
```

`patchMaterial` 做的事情非常少，但非常关键：它给材质的 `userData` 增加 `outlineParameters`。

默认参数：

- `thickness: 0.008`：描边厚度。
- `color: [0, 0, 0]`：黑色描边。

这些值不会被普通 Three.js 渲染器使用，而是被本项目的 `OutlineEffect` 读取。

### 4.2 setSelected

关键位置：

- `src/js/shot-generator/helpers/outlineMaterial.js:15` 到 `src/js/shot-generator/helpers/outlineMaterial.js:29`

```js
if (material.userData.outlineParameters) {
  material.userData.outlineParameters.color = selected ? SELECTED_COLOR : DEFAULT_COLOR
  material.color.set(blocked ? 0x888888 : defaultColor)
  material.needsUpdate = true
}
```

`setSelected` 的效果：

- 未选中：描边颜色是黑色。
- 选中：描边颜色变成偏紫色的 `SELECTED_COLOR`。
- blocked：基础材质颜色变灰。
- 非 blocked：基础材质颜色设置为调用方传入的默认色。

人物角色在 `src/js/shot-generator/components/Three/Character.js:391` 到 `src/js/shot-generator/components/Three/Character.js:394` 遍历 LOD 下所有 mesh 调用：

```js
setSelected(child, isSelected, false, 0xffffff)
```

因此，截图中未选中的白色人物会是：

- 基础材质颜色：白色 `0xffffff`
- 轮廓材质颜色：黑色 `[0, 0, 0]`

## 5. Shading Mode 如何选择 OutlineEffect

### 5.1 默认世界渲染模式

文件：`src/js/shared/reducers/shot-generator.js`

默认场景的 `world.shadingMode` 是 `ShadingType.Outline`：

- `src/js/shared/reducers/shot-generator.js:594`

```js
shadingMode: ShadingType.Outline
```

所以新建 Shot Generator 场景时，主视图默认就是带描边的 Outline 模式。

### 5.2 渲染模式枚举

文件：`src/js/vendor/shading-effects/ShadingType.js`

`ShadingType` 包含四种模式：

```js
Outline
Wireframe
Flat
Depth
```

`CYCLE_SHADING_MODE` 会按 `Object.values(ShadingType)` 顺序循环切换：

- `src/js/shared/reducers/shot-generator.js:1372` 到 `src/js/shared/reducers/shot-generator.js:1389`

如果 `shadingMode` 缺失或非法，会回退到 `Outline`。

### 5.3 createShadingEffect

文件：`src/js/vendor/shading-effects/createShadingEffect.js`

关键位置：

- `src/js/vendor/shading-effects/createShadingEffect.js:7` 到 `src/js/vendor/shading-effects/createShadingEffect.js:24`

```js
case ShadingType.Outline:
default:
  newRenderer = new OutlineEffect(gl, { defaultThickness: 0.015 })
  break
```

注意这里传入的 `defaultThickness: 0.015` 只是 `OutlineEffect` 的全局默认值。对于人物材质，因为 `patchMaterial` 设置了 `outlineParameters.thickness = 0.008`，最终会优先使用材质级参数。

### 5.4 useShadingEffect

文件：`src/js/shot-generator/hooks/use-shading-effect.js`

`useShadingEffect` 会为同一个底层 `gl` 创建四种 effect 实例：

- `src/js/shot-generator/hooks/use-shading-effect.js:6` 到 `src/js/shot-generator/hooks/use-shading-effect.js:11`

然后在 `shadingMode` 变化时切换当前 renderer 引用：

- `src/js/shot-generator/hooks/use-shading-effect.js:38` 到 `src/js/shot-generator/hooks/use-shading-effect.js:41`

```js
renderer.current = instances.current[shadingMode]
```

这意味着 UI 切换阴影模式时，不是重新创建 WebGLRenderer，而是在四个 effect 包装器之间切换。

### 5.5 主视图每帧渲染

文件：`src/js/shot-generator/SceneManagerR3fLarge.js`

关键位置：

- `src/js/shot-generator/SceneManagerR3fLarge.js:323` 到 `src/js/shot-generator/SceneManagerR3fLarge.js:338`

```js
const renderer = useShadingEffect(
  gl,
  mainViewCamera === 'live' ? world.shadingMode : ShadingType.Outline,
  world.backgroundColor
)

useFrame(({ scene, camera }) => {
  renderer.current.render(scene, camera)
})
```

含义：

- 主视图是 `live` 相机时，使用 `world.shadingMode`。
- 主视图不是 `live` 时，强制使用 `ShadingType.Outline`。
- 每帧都调用当前 effect 的 `render` 方法。

因此，截图中的人物黑边来自 `renderer.current` 指向的 `OutlineEffect`。

## 6. OutlineEffect 的两遍渲染原理

文件：`src/js/vendor/OutlineEffect.js`

`OutlineEffect` 是一个 renderer 包装器。它不改变业务组件结构，而是在 render 时临时替换材质进行第二遍绘制。

### 6.1 初始化默认参数

关键位置：

- `src/js/vendor/OutlineEffect.js:64` 到 `src/js/vendor/OutlineEffect.js:109`

主要默认值：

```js
defaultThickness = 0.003 或传入值
defaultColor = [0, 0, 0]
defaultAlpha = 1.0
```

同时初始化：

- `cache`：原始材质 uuid -> 描边材质。
- `originalMaterials`：描边材质 uuid -> 原始材质。
- `originalOnBeforeRenders`：临时保存 mesh 的 `onBeforeRender`。

### 6.2 描边 Shader 的顶点扩张

关键位置：

- `src/js/vendor/OutlineEffect.js:112` 到 `src/js/vendor/OutlineEffect.js:155`

描边 shader 注入了 `calculateOutline`：

```glsl
vec4 calculateOutline( vec4 pos, vec3 objectNormal, vec4 skinned ) {
  float thickness = outlineThickness;
  float aspect = projectionMatrix[0][0]/projectionMatrix[1][1];
  vec4 pos2 = projectionMatrix * modelViewMatrix * vec4( skinned.xyz + objectNormal, 1.0 );
  vec4 norm = -normalize( pos - pos2 );
  return pos + norm * thickness * pos.w * ratio * vec4(aspect,1.0,1.0,1.0);
}
```

核心思想：

1. 计算顶点沿法线偏移后的投影位置。
2. 得到屏幕空间中的外扩方向 `norm`。
3. 按 `outlineThickness` 把顶点向外推。
4. 用 `aspect` 修正横纵比例，避免宽屏下描边厚度变形。

然后在原 vertex shader 的 `main` 结尾前插入：

```glsl
gl_Position = calculateOutline(gl_Position, objectNormal, vec4(transformed, 1.0));
```

所以黑边实际是一个“被顶点法线外扩的背面壳层”。

### 6.3 描边材质使用 BackSide

关键位置：

- `src/js/vendor/OutlineEffect.js:220` 到 `src/js/vendor/OutlineEffect.js:247`

`createMaterial` 创建 `THREE.ShaderMaterial`，其中最重要的是：

```js
side: THREE.BackSide
```

为什么是 `BackSide`：

- 如果画正面，外扩壳层会覆盖原模型表面。
- 只画背面时，外扩后的壳层只会从模型边缘露出来。
- 原模型第一遍正常渲染在前，第二遍背面壳层在边缘可见，就形成黑色轮廓。

这也是卡通描边常见的 inverted hull / backface outline 技术。

### 6.4 哪些材质会有描边

关键位置：

- `src/js/vendor/OutlineEffect.js:251` 到 `src/js/vendor/OutlineEffect.js:279`

```js
if (originalMaterial.userData.outlineParameters) {
  data = {
    material: createMaterial(originalMaterial),
    used: true,
    keepAlive: defaultKeepAlive,
    count: 0
  }
} else {
  data = {
    material: createInvisibleMaterial(),
    used: true,
    keepAlive: defaultKeepAlive,
    count: 0
  }
}
```

这段逻辑非常关键：

- 有 `outlineParameters` 的材质，会创建可见描边材质。
- 没有 `outlineParameters` 的材质，会创建 invisible material。

因此，不是所有 Three.js mesh 都会出现黑边。人物之所以有黑边，是因为 `Character.js` 对人物材质调用了 `patchMaterial`。

### 6.5 临时替换材质

关键位置：

- `src/js/vendor/OutlineEffect.js:295` 到 `src/js/vendor/OutlineEffect.js:319`
- `src/js/vendor/OutlineEffect.js:321` 到 `src/js/vendor/OutlineEffect.js:343`

`setOutlineMaterial` 会在第二遍渲染前遍历场景，把 mesh 的材质替换为描边材质；`restoreOriginalMaterial` 在第二遍渲染后还原。

这使得描边实现不会永久污染业务对象的材质引用。

### 6.6 同步材质级描边参数

关键位置：

- `src/js/vendor/OutlineEffect.js:356` 到 `src/js/vendor/OutlineEffect.js:408`

`updateUniforms` 读取原材质的：

- `outlineParameters.thickness`
- `outlineParameters.color`
- `outlineParameters.alpha`

并写入描边材质 uniform。

`updateOutlineMaterial` 同步原材质的：

- `skinning`
- `morphTargets`
- `morphNormals`
- `fog`
- `visible`
- `transparent`

这对人物模型很重要，因为人物是 `SkinnedMesh`，需要骨骼动画 / 姿态变形支持。`OutlineEffect` 在描边材质上同步 `skinning`，才能让描边跟着人物姿态变化。

### 6.7 两遍 render

关键位置：

- `src/js/vendor/OutlineEffect.js:460` 到 `src/js/vendor/OutlineEffect.js:525`

`render(scene, camera)` 执行顺序：

```text
1. renderer.render(scene, camera)
   正常渲染原场景。

2. this.renderOutline(scene, camera)
   进入描边渲染。

3. renderOutline 内部：
   - scene.background = null
   - renderer.autoClear = false
   - renderer.shadowMap.enabled = false
   - scene.traverse(setOutlineMaterial)
   - renderer.render(scene, camera)
   - scene.traverse(restoreOriginalMaterial)
   - cleanupCache()
   - 恢复 scene / renderer 状态
```

`renderer.autoClear = false` 保证第二遍描边不会清空第一遍正常渲染结果；`scene.background = null` 保证第二遍不会覆盖背景。

## 7. 人物黑边与物体 / 道具黑边的共用机制

虽然本问题关注人物模型，但项目中很多 3D 对象都使用同一套机制：

### 7.1 普通物体

文件：`src/js/shot-generator/components/Three/ModelObject.js`

普通物体的 `materialFactory` 也创建 `MeshToonMaterial` 并调用 `patchMaterial`：

- `src/js/shot-generator/components/Three/ModelObject.js:17` 到 `src/js/shot-generator/components/Three/ModelObject.js:25`

选中状态通过 `setSelected` 更新：

- `src/js/shot-generator/components/Three/ModelObject.js:94` 到 `src/js/shot-generator/components/Three/ModelObject.js:100`

### 7.2 可附着道具

文件：`src/js/shot-generator/components/Three/Attachable.js`

Attachable 的 `materialFactory` 同样调用 `patchMaterial`：

- `src/js/shot-generator/components/Three/Attachable.js:11` 到 `src/js/shot-generator/components/Three/Attachable.js:20`

因此，“黑边风格”是 Shot Generator 里一套通用渲染风格，不是人物独有逻辑。

## 8. 与截图效果的对应关系

截图中可以看到：

- 人物外轮廓是粗黑线。
- 身体内部也有少量黑线 / 灰线。
- 人物整体是白色卡通材质。
- 背景和地面网格没有同样粗的外扩轮廓。

对应代码解释：

1. 外轮廓黑线来自 `OutlineEffect` 第二遍 BackSide 壳层渲染。
2. 人物白色基础来自 `Character.js` 创建的 `MeshToonMaterial({ color: 0xffffff })`。
3. 选中时轮廓可变成偏紫色，来自 `setSelected` 的 `SELECTED_COLOR`。
4. 非 `patchMaterial` 的材质不会得到可见 outline；没有 `outlineParameters` 时 `OutlineEffect` 会创建 invisible material。
5. 内部灰白明暗主要来自 toon 材质、贴图、光照和模型自身几何/法线，不是 `OutlineEffect` 的边缘检测线。

## 9. 关键数据结构

### 9.1 材质上的 outlineParameters

```js
material.userData.outlineParameters = {
  thickness: 0.008,
  color: [0.0, 0.0, 0.0]
}
```

可选字段还包括：

- `alpha`
- `visible`
- `keepAlive`

这些字段由 `OutlineEffect.updateUniforms` 和 `OutlineEffect.updateOutlineMaterial` 读取。

### 9.2 OutlineEffect 缓存结构

```js
cache[originalMaterial.uuid] = {
  material: outlineShaderMaterial,
  used: true,
  keepAlive: defaultKeepAlive,
  count: 0
}
```

缓存的作用是避免每帧都重新创建描边材质。`cleanupCache` 会回收长时间未使用且没有 keepAlive 的描边材质。

## 10. 调整黑边效果的入口

如果后续要改人物黑边效果，主要有这些入口：

| 需求 | 修改位置 |
| --- | --- |
| 改人物默认黑边粗细 | `src/js/shot-generator/helpers/outlineMaterial.js:7` 的 `thickness: 0.008` |
| 改人物默认黑边颜色 | `src/js/shot-generator/helpers/outlineMaterial.js:3` 的 `DEFAULT_COLOR` |
| 改人物选中描边颜色 | `src/js/shot-generator/helpers/outlineMaterial.js:2` 的 `SELECTED_COLOR` |
| 给某一类对象定制粗细 | 调用 `patchMaterial(material, { thickness: ... })` |
| 改 Outline 模式全局默认粗细 | `src/js/vendor/shading-effects/createShadingEffect.js:21` 的 `defaultThickness: 0.015` |
| 改描边算法 | `src/js/vendor/OutlineEffect.js` 中 `vertexShaderChunk` / `fragmentShader` |
| 关闭某材质描边 | 不调用 `patchMaterial`，或设置 `outlineParameters.visible = false` |

## 11. 潜在注意点

1. `OutlineEffect` 是两遍渲染，Outline 模式比普通渲染更耗性能。
2. 描边依赖 mesh normal；模型法线异常时，黑边可能破碎或厚薄不一致。
3. 人物是 `SkinnedMesh`，描边材质必须同步 `skinning`，否则轮廓不会跟随骨骼姿态。
4. `patchMaterial` 是是否可描边的开关；如果导入的新模型没有走 `Character.js` / `ModelObject.js` / `Attachable.js` 的材质替换流程，可能没有黑边。
5. 轮廓线是背面外扩壳层，不是屏幕空间边缘检测，因此内部轮廓线不会像 Sobel 边缘检测那样自动识别所有深度/法线突变。
6. `OutlineEffect` 会临时替换材质再还原；如果某些自定义材质依赖复杂 `onBeforeRender` 或 shader 特性，可能需要单独验证。

## 12. 总结

Shot Generator 中人物边缘黑边由三部分共同实现：

1. `Character.js` 把人物 mesh 材质替换为 `MeshToonMaterial`，并通过 `patchMaterial` 添加 `outlineParameters`。
2. `world.shadingMode` 默认是 `Outline`，`SceneManagerR3fLarge` 每帧通过 `useShadingEffect` 使用 `OutlineEffect` 渲染场景。
3. `OutlineEffect` 先正常渲染原模型，再用 BackSide 的描边 shader 渲染一次沿法线外扩的黑色壳层，形成截图中的人物外轮廓。

因此，黑边的核心代码入口是：

- 材质标记：`src/js/shot-generator/helpers/outlineMaterial.js`
- 人物材质创建：`src/js/shot-generator/components/Three/Character.js`
- 渲染模式选择：`src/js/shot-generator/hooks/use-shading-effect.js`
- 描边渲染算法：`src/js/vendor/OutlineEffect.js`
