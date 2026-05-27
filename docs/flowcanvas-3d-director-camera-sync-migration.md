# FlowCanvas 3D 导演台：相机预览框与主场景双向同步移植文档

## 1. 功能目标

在 FlowCanvas 的 3D 导演台中，实现与 Storyboarder Shot Generator 一致的相机控制体验：

- 左上角提供俯视预览框，展示人物、物体、灯光、相机的位置。
- 用户可以在左上角拖动相机图标，调整相机与人物/物体的距离和角度。
- 右侧主 3D 场景实时显示该相机看到的透视画面。
- 用户也可以在右侧主 3D 场景中直接调整相机视角、位置、镜头参数。
- 右侧调整时，左上角相机图标、视锥线、位置也要实时更新。
- 多相机情况下，左上角点击相机图标可以切换 active camera。
- 相机参数面板需要与两个视图保持同步。

最终效果：

```text
左上角俯视预览框拖动 Camera
→ 更新相机位置
→ 右侧主场景实时变化

右侧主场景操作相机
→ 主 Three.js camera 实时变化
→ 左上角 CameraIcon 实时跟随
→ 交互结束后写回状态
```

## 2. 原项目核心参考文件

### 视图布局

- `src/js/shot-generator/components/Editor/index.js`
  - 左上角小 Canvas：`SceneManagerR3fSmall`
  - 右侧主 Canvas：`SceneManagerR3fLarge`
  - 保存两个 Canvas 的 `camera / scene / gl` 引用

### 右侧主场景

- `src/js/shot-generator/SceneManagerR3fLarge.js`
  - 渲染主 3D 场景
  - 挂载 `CameraUpdate`
  - 挂载 `InteractionManager`
  - 负责把 Redux 中的 active camera 应用到 Three.js camera

### 左上角俯视预览

- `src/js/shot-generator/SceneManagerR3fSmall.js`
  - 渲染俯视小地图
  - 渲染 CameraIcon
  - 处理拖动相机图标
  - 拖动时更新状态

### 相机图标与视锥

- `src/js/shot-generator/components/Three/Icons/CameraIcon.js`
  - 显示相机图标
  - 显示相机名称、焦距、高度
  - 显示相机视锥线
  - 每帧读取主 camera 状态并同步图标

### 主相机同步

- `src/js/shot-generator/CameraUpdate.js`
  - 从状态读取 active camera
  - 同步到 Three.js PerspectiveCamera

### 主场景相机交互

- `src/js/shot-generator/components/Three/InteractionManager.js`
  - 处理右侧主 Canvas 的 pointer 事件
  - 把事件传给 `CameraControlsComponent`

- `src/js/shot-generator/components/Three/CameraControlsComponet.js`
  - 创建 `CameraControls`
  - 拖动过程中直接更新 Three.js camera
  - 交互结束后写回状态

- `src/js/shot-generator/CameraControls.js`
  - 自定义相机控制逻辑
  - 负责 orbit / pan / move / tilt / roll / fov 等计算

### 拖动物体 / 相机图标

- `src/js/shot-generator/hooks/use-dragging-manager.js`
  - 左上角拖动相机图标时计算平面交点
  - 更新相机 `x / y` 位置

### 状态管理

- `src/js/shared/reducers/shot-generator.js`
  - `sceneObjects`
  - `activeCamera`
  - `updateObject`
  - `updateObjects`
  - `setActiveCamera`

## 3. 推荐在 FlowCanvas 中抽象出的模块

建议不要直接把 Storyboarder 的旧 Redux 代码原封不动搬过去，而是按功能拆成以下模块。

```text
3d-director/
├── state/
│   └── directorStore.ts
├── types/
│   └── directorTypes.ts
├── components/
│   ├── DirectorStage.tsx
│   ├── MainViewport.tsx
│   ├── TopdownViewport.tsx
│   ├── CameraIcon.tsx
│   ├── CameraSync.tsx
│   ├── CameraControlsBridge.tsx
│   └── CameraInspector.tsx
├── hooks/
│   ├── useTopdownDrag.ts
│   ├── useCameraSync.ts
│   └── useDirectorCanvasRefs.ts
└── utils/
    ├── cameraMapping.ts
    ├── cameraMath.ts
    └── frustum.ts
```

如果 FlowCanvas 已经有 3D 导演台目录，则按现有结构合并即可。

## 4. 核心状态设计

Storyboarder 中相机是 `sceneObjects` 的一种对象。FlowCanvas 建议也保持类似设计。

### Camera 数据结构

```ts
export type DirectorObjectType =
  | 'camera'
  | 'character'
  | 'object'
  | 'light'
  | 'image'
  | 'volume'

export interface DirectorCamera {
  id: string
  type: 'camera'
  name?: string
  visible: boolean
  locked?: boolean

  // 地面平面位置
  x: number
  y: number

  // 高度
  z: number

  // 水平旋转，单位 rad
  rotation: number

  // 上下俯仰，单位 rad
  tilt: number

  // 滚转，单位 rad
  roll: number

  // Three.js PerspectiveCamera.fov，单位 degree
  fov: number
}
```

### 全局导演台状态

```ts
export interface DirectorState {
  objects: Record<string, DirectorObject>
  activeCameraId: string | null
  selectedIds: string[]

  world: {
    backgroundColor: number
    groundVisible: boolean
    roomVisible: boolean
  }
}
```

### 必须提供的状态操作

```ts
export interface DirectorActions {
  setActiveCamera: (cameraId: string) => void
  selectObject: (id: string | null) => void

  updateObject: <T extends DirectorObject>(
    id: string,
    patch: Partial<T>
  ) => void

  updateObjects: (
    patches: Record<string, Partial<DirectorObject>>
  ) => void
}
```

如果 FlowCanvas 使用 Zustand，推荐：

```ts
export const useDirectorStore = create<DirectorState & DirectorActions>((set) => ({
  objects: {},
  activeCameraId: null,
  selectedIds: [],

  world: {
    backgroundColor: 0xe5e5e5,
    groundVisible: true,
    roomVisible: false,
  },

  setActiveCamera: (cameraId) => set({ activeCameraId: cameraId }),

  selectObject: (id) => set({ selectedIds: id ? [id] : [] }),

  updateObject: (id, patch) => set((state) => ({
    objects: {
      ...state.objects,
      [id]: {
        ...state.objects[id],
        ...patch,
      },
    },
  })),

  updateObjects: (patches) => set((state) => {
    const nextObjects = { ...state.objects }

    for (const [id, patch] of Object.entries(patches)) {
      const current = nextObjects[id]
      if (!current || current.locked) continue

      nextObjects[id] = {
        ...current,
        ...patch,
      }
    }

    return { objects: nextObjects }
  }),
}))
```

## 5. 坐标轴映射规则

这是移植时最容易出错的地方。

Storyboarder 的业务状态与 Three.js 坐标不是同名映射：

```text
业务状态 x → Three.js position.x
业务状态 y → Three.js position.z
业务状态 z → Three.js position.y
```

原因：

- Three.js 默认 `Y` 是高度轴。
- Storyboarder / Shot Generator 业务数据里 `Z` 表示高度。
- 业务数据里的 `Y` 表示地面平面上的前后方向。

FlowCanvas 如果想 1:1 移植，建议保留这个映射，避免后续相机、人物、地面、俯视图逻辑全部反向重写。

建议封装工具函数：

```ts
import * as THREE from 'three'

export interface DirectorTransform {
  x: number
  y: number
  z: number
}

export function statePositionToThree(position: DirectorTransform): THREE.Vector3 {
  return new THREE.Vector3(position.x, position.z, position.y)
}

export function threePositionToState(position: THREE.Vector3): DirectorTransform {
  return {
    x: position.x,
    y: position.z,
    z: position.y,
  }
}
```

相机旋转同步：

```ts
export function applyCameraStateToThreeCamera(
  camera: THREE.PerspectiveCamera,
  cameraState: DirectorCamera
) {
  camera.position.set(
    cameraState.x,
    cameraState.z,
    cameraState.y
  )

  camera.rotation.set(0, 0, 0)
  camera.rotation.y = cameraState.rotation
  camera.rotateX(cameraState.tilt)
  camera.rotateZ(cameraState.roll)

  camera.fov = cameraState.fov
  camera.updateProjectionMatrix()

  camera.userData.type = 'camera'
  camera.userData.id = cameraState.id
  camera.userData.locked = cameraState.locked
}
```

从 Three.js camera 反推业务状态：

```ts
export function threeCameraToCameraState(
  camera: THREE.PerspectiveCamera
): Pick<DirectorCamera, 'x' | 'y' | 'z' | 'rotation' | 'tilt' | 'roll' | 'fov'> {
  const euler = new THREE.Euler().setFromQuaternion(camera.quaternion, 'YXZ')

  return {
    x: camera.position.x,
    y: camera.position.z,
    z: camera.position.y,
    rotation: euler.y,
    tilt: euler.x,
    roll: euler.z,
    fov: camera.fov,
  }
}
```

## 6. 双 Canvas 架构

必须保留两个独立 Canvas：

```text
DirectorStage
├── TopdownViewport     左上角俯视预览框
└── MainViewport        右侧主 3D 视图
```

它们共享同一份 director store。

### DirectorStage 示例

```tsx
export function DirectorStage() {
  const mainCanvasRef = useRef<DirectorCanvasData | null>(null)
  const topdownCanvasRef = useRef<DirectorCanvasData | null>(null)

  return (
    <div className="director-stage">
      <div className="director-topdown">
        <TopdownViewport
          mainCanvasRef={mainCanvasRef}
          topdownCanvasRef={topdownCanvasRef}
        />
      </div>

      <div className="director-main">
        <MainViewport
          mainCanvasRef={mainCanvasRef}
          topdownCanvasRef={topdownCanvasRef}
        />
      </div>
    </div>
  )
}
```

### CanvasData 类型

```ts
export interface DirectorCanvasData {
  camera: THREE.Camera
  scene: THREE.Scene
  gl: THREE.WebGLRenderer
}
```

主 Canvas 需要把自己的 `camera / scene / gl` 暴露给左上角 `CameraIcon` 使用。

## 7. 右侧主视图：MainViewport

`MainViewport` 负责：

- 渲染 3D 人物、物体、地面、灯光。
- 使用 active camera 作为真实拍摄相机。
- 状态变化时同步 Three.js camera。
- 用户操作时实时更新 camera。
- 交互结束后写回状态。

核心组件：

```tsx
export function MainViewport(props: MainViewportProps) {
  return (
    <Canvas>
      <MainSceneContent {...props} />
    </Canvas>
  )
}
```

### MainSceneContent 结构

```tsx
function MainSceneContent({
  mainCanvasRef,
  topdownCanvasRef,
}: MainSceneContentProps) {
  const { scene, camera, gl } = useThree()

  useEffect(() => {
    mainCanvasRef.current = { scene, camera, gl }
  }, [scene, camera, gl])

  return (
    <>
      <CameraSync />
      <CameraControlsBridge mainCanvasRef={mainCanvasRef} />
      <WorldRenderer />
      <ObjectRenderer />
      <CharacterRenderer />
      <LightRenderer />
    </>
  )
}
```

## 8. CameraSync：状态 → Three.js camera

对应 Storyboarder：

`src/js/shot-generator/CameraUpdate.js`

FlowCanvas 中建议实现为：

```tsx
export function CameraSync() {
  const { camera } = useThree()

  const activeCamera = useDirectorStore((state) => {
    const id = state.activeCameraId
    if (!id) return null

    const object = state.objects[id]
    return object?.type === 'camera' ? object : null
  })

  useEffect(() => {
    if (!activeCamera) return
    if (!(camera instanceof THREE.PerspectiveCamera)) return

    applyCameraStateToThreeCamera(camera, activeCamera)

    camera.userData.isSynchronized = true
  }, [camera, activeCamera])

  return null
}
```

作用：

```text
Redux/Zustand activeCamera 变化
→ CameraSync useEffect 触发
→ Three.js PerspectiveCamera 更新
→ 右侧主视图画面变化
```

## 9. 左上角俯视图：TopdownViewport

`TopdownViewport` 负责：

- 使用 OrthographicCamera 俯视整个场景。
- 渲染所有对象的平面位置。
- 相机渲染为 CameraIcon。
- 相机图标可拖拽。
- 拖拽时写回状态。

```tsx
export function TopdownViewport({
  mainCanvasRef,
  topdownCanvasRef,
}: TopdownViewportProps) {
  return (
    <Canvas orthographic>
      <TopdownSceneContent
        mainCanvasRef={mainCanvasRef}
        topdownCanvasRef={topdownCanvasRef}
      />
    </Canvas>
  )
}
```

### TopdownSceneContent

```tsx
function TopdownSceneContent({
  mainCanvasRef,
  topdownCanvasRef,
}: TopdownSceneContentProps) {
  const { scene, camera, gl } = useThree()

  useEffect(() => {
    topdownCanvasRef.current = { scene, camera, gl }
  }, [scene, camera, gl])

  const objects = useDirectorStore((state) => state.objects)
  const cameras = Object.values(objects).filter(
    (object): object is DirectorCamera => object.type === 'camera'
  )

  return (
    <>
      <TopdownCameraSetup />
      <TopdownObjectIcons />

      {cameras.map((cameraObject) => (
        <CameraIcon
          key={cameraObject.id}
          cameraObject={cameraObject}
          mainCameraRef={mainCanvasRef}
        />
      ))}
    </>
  )
}
```

## 10. Topdown 相机设置

左上角小地图建议使用 OrthographicCamera，俯视整个地面平面。

Storyboarder 参考：

`src/js/shot-generator/SceneManagerR3fSmall.js:204`

```js
camera.position.y = 900
camera.rotation.x = -Math.PI / 2
camera.layers.enable(2)
camera.updateMatrixWorld(true)
```

FlowCanvas 实现：

```tsx
function TopdownCameraSetup() {
  const { camera } = useThree()

  useEffect(() => {
    camera.position.set(0, 900, 0)
    camera.rotation.set(-Math.PI / 2, 0, 0)
    camera.near = -1000
    camera.far = 1000
    camera.updateProjectionMatrix()
    camera.updateMatrixWorld(true)
  }, [camera])

  return null
}
```

## 11. CameraIcon：相机图标与视锥

`CameraIcon` 要实现三件事：

1. 根据 camera state 显示位置。
2. 根据 `rotation / fov` 显示视锥。
3. 当右侧主 camera 正在被操作时，每帧读取主 camera 并同步小图标。

Storyboarder 参考：

`src/js/shot-generator/components/Three/Icons/CameraIcon.js`

### 基础位置

```tsx
export function CameraIcon({
  cameraObject,
  mainCameraRef,
}: CameraIconProps) {
  const groupRef = useRef<THREE.Group>(null)
  const setActiveCamera = useDirectorStore((state) => state.setActiveCamera)
  const selectObject = useDirectorStore((state) => state.selectObject)

  useEffect(() => {
    if (!groupRef.current) return

    groupRef.current.position.set(
      cameraObject.x,
      cameraObject.z,
      cameraObject.y
    )

    groupRef.current.rotation.y = cameraObject.rotation
  }, [cameraObject])

  return (
    <group
      ref={groupRef}
      userData={{
        id: cameraObject.id,
        type: 'camera',
        locked: cameraObject.locked,
      }}
      onPointerDown={(event) => {
        event.stopPropagation()
        selectObject(cameraObject.id)
        setActiveCamera(cameraObject.id)
      }}
    >
      <CameraBody />
      <CameraFrustum cameraObject={cameraObject} />
      <CameraLabel cameraObject={cameraObject} />
    </group>
  )
}
```

### 视锥计算

Storyboarder 里计算 hFOV 的逻辑：

```js
let hFOV = 2 * Math.atan(Math.tan(sceneObject.fov * Math.PI / 180 / 2) * 1)
```

FlowCanvas 推荐封装：

```ts
export function getHorizontalFovRad(
  verticalFovDeg: number,
  aspect = 1
) {
  return 2 * Math.atan(
    Math.tan(THREE.MathUtils.degToRad(verticalFovDeg) / 2) * aspect
  )
}
```

视锥渲染可以用 `lineSegments`：

```tsx
function CameraFrustum({ cameraObject }: { cameraObject: DirectorCamera }) {
  const hFov = getHorizontalFovRad(cameraObject.fov, 1)
  const length = 2.5

  const leftAngle = cameraObject.rotation + hFov / 2
  const rightAngle = cameraObject.rotation - hFov / 2

  const points = useMemo(() => {
    const origin = new THREE.Vector3(0, 0, 0)

    const left = new THREE.Vector3(
      Math.sin(leftAngle) * length,
      0,
      Math.cos(leftAngle) * length
    )

    const right = new THREE.Vector3(
      Math.sin(rightAngle) * length,
      0,
      Math.cos(rightAngle) * length
    )

    return [origin, left, origin, right]
  }, [leftAngle, rightAngle])

  return (
    <lineSegments>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          args={[new Float32Array(points.flatMap((p) => [p.x, p.y, p.z])), 3]}
        />
      </bufferGeometry>
      <lineBasicMaterial color="black" />
    </lineSegments>
  )
}
```

## 12. 左上角拖动 CameraIcon

Storyboarder 参考：

- `src/js/shot-generator/SceneManagerR3fSmall.js:96`
- `src/js/shot-generator/hooks/use-dragging-manager.js`

FlowCanvas 推荐实现 `useTopdownDrag`。

### 核心算法

```text
pointerdown
→ raycaster 从 topdown camera 发射射线
→ 与拖动平面求交
→ 记录相机当前位置与交点 offset

pointermove
→ 再次求交
→ newPosition = intersection - offset
→ 更新 CameraIcon 的 Three.js 位置
→ updateObject 写回 director store

pointerup
→ 最后一次 updateObject
→ 清理 drag 状态
```

### Hook 示例

```ts
export function useTopdownDrag() {
  const raycasterRef = useRef(new THREE.Raycaster())
  const planeRef = useRef(new THREE.Plane())
  const intersectionRef = useRef(new THREE.Vector3())
  const offsetRef = useRef(new THREE.Vector3())
  const targetRef = useRef<THREE.Object3D | null>(null)

  const updateObject = useDirectorStore((state) => state.updateObject)

  const beginDrag = useCallback((
    event: ThreeEvent<PointerEvent>,
    target: THREE.Object3D
  ) => {
    const camera = event.camera
    targetRef.current = target

    raycasterRef.current.setFromCamera(event.pointer, camera)

    // 俯视图拖动使用水平地面平面
    planeRef.current.setFromNormalAndCoplanarPoint(
      camera.position.clone().normalize(),
      target.position
    )

    if (raycasterRef.current.ray.intersectPlane(
      planeRef.current,
      intersectionRef.current
    )) {
      offsetRef.current.copy(intersectionRef.current).sub(target.position)
    }
  }, [])

  const drag = useCallback((event: ThreeEvent<PointerEvent>) => {
    const target = targetRef.current
    if (!target) return

    raycasterRef.current.setFromCamera(event.pointer, event.camera)

    if (!raycasterRef.current.ray.intersectPlane(
      planeRef.current,
      intersectionRef.current
    )) {
      return
    }

    const next = intersectionRef.current.clone().sub(offsetRef.current)

    // 俯视图只改地面平面 x/z，不改高度 y
    target.position.set(next.x, target.position.y, next.z)

    const id = target.userData.id
    if (!id) return

    updateObject(id, {
      x: next.x,
      y: next.z,
    })
  }, [updateObject])

  const endDrag = useCallback(() => {
    targetRef.current = null
  }, [])

  return {
    beginDrag,
    drag,
    endDrag,
  }
}
```

注意：这里写回的是：

```ts
updateObject(id, {
  x: next.x,
  y: next.z,
})
```

不是写 `z`。因为业务状态中的 `z` 是高度，不能被俯视图拖动改变。

## 13. 右侧主场景相机控制

右侧主场景需要实现两阶段同步。

### 阶段 1：拖动过程中直接更新 Three.js camera

这样保证流畅，不需要每一帧都走状态管理。

```text
用户拖动右侧主场景
→ 直接修改 Three.js camera.position / rotation / fov
→ camera.userData.isSynchronized = false
→ 左上角 CameraIcon 每帧读取 mainCamera 并同步
```

### 阶段 2：交互结束后写回状态

```text
pointerup / control end
→ 从 Three.js camera 反推 camera state
→ updateObject(activeCameraId, cameraState)
```

Storyboarder 参考：

`src/js/shot-generator/components/Three/CameraControlsComponet.js:69`

```js
const onCameraUpdate = ({active, object}) => {
  if (!active) {
    updateObject(camera.userData.id, {
      x: object.x,
      y: object.y,
      z: object.z,
      rotation: object.rotation,
      tilt: object.tilt,
      roll: object.roll,
      fov: object.fov
    })
  } else {
    camera.position.x = object.x
    camera.position.y = object.z
    camera.position.z = object.y
    camera.rotation.x = 0
    camera.rotation.z = 0
    camera.rotation.y = object.rotation
    camera.rotateX(object.tilt)
    camera.rotateZ(object.roll)
    camera.fov = object.fov
    camera.updateProjectionMatrix()
    camera.isSynchronized = false
  }
}
```

FlowCanvas 可以做成：

```tsx
export function CameraControlsBridge() {
  const { camera, gl } = useThree()

  const activeCamera = useDirectorStore((state) => {
    if (!state.activeCameraId) return null
    const object = state.objects[state.activeCameraId]
    return object?.type === 'camera' ? object : null
  })

  const updateObject = useDirectorStore((state) => state.updateObject)

  const controlsRef = useRef<DirectorCameraControls | null>(null)

  useEffect(() => {
    if (!activeCamera) return
    if (!(camera instanceof THREE.PerspectiveCamera)) return

    controlsRef.current = new DirectorCameraControls(camera, gl.domElement)

    controlsRef.current.onChange = (isActive) => {
      if (isActive) {
        camera.userData.isSynchronized = false
      } else {
        const nextCameraState = threeCameraToCameraState(camera)

        updateObject(activeCamera.id, nextCameraState)

        camera.userData.isSynchronized = true
      }
    }

    return () => {
      controlsRef.current?.dispose()
      controlsRef.current = null
    }
  }, [camera, gl.domElement, activeCamera?.id, updateObject])

  useFrame((_, delta) => {
    controlsRef.current?.update(delta)
  })

  return null
}
```

`DirectorCameraControls` 可以先移植 Storyboarder 的 `CameraControls.js`，再逐步 TypeScript 化。

## 14. 右侧主视图 → 左上角实时同步

这是 1:1 体验的关键。

仅靠状态管理是不够的，因为右侧主视图拖动过程中可能不每帧写状态。Storyboarder 的做法是让左上角 `CameraIcon` 每帧读取右侧真实 camera。

参考：

`src/js/shot-generator/components/Three/Icons/CameraIcon.js:47`

```js
useFrame(() => {
  if (!mainCamera || sceneObject.id !== mainCamera.userData.id || !iconsSprites.current || !fontMesh || mainCamera.isSynchronized) return false

  const currentRotation = new THREE.Euler().setFromQuaternion(mainCamera.quaternion, 'YXZ').y

  iconsSprites.current.icon.material.rotation = currentRotation

  let hFOV = 2 * Math.atan(Math.tan(mainCamera.fov * Math.PI / 180 / 2) * 1)

  frustumIcons.current.left.icon.material.rotation = hFOV/2 + currentRotation
  frustumIcons.current.right.icon.material.rotation = -hFOV/2 + currentRotation

  let isPositionChanged = !ref.current.position.equals(mainCamera.position)

  if(isPositionChanged) {
    ref.current.position.copy(mainCamera.position)
    props.autofitOrtho()
  }

  mainCamera.isSynchronized = true
})
```

FlowCanvas 实现：

```tsx
function useSyncCameraIconFromMainCamera(
  groupRef: React.RefObject<THREE.Group>,
  cameraObject: DirectorCamera,
  mainCanvasRef: React.RefObject<DirectorCanvasData | null>
) {
  useFrame(() => {
    const mainCamera = mainCanvasRef.current?.camera

    if (!(mainCamera instanceof THREE.PerspectiveCamera)) return
    if (!groupRef.current) return
    if (mainCamera.userData.id !== cameraObject.id) return
    if (mainCamera.userData.isSynchronized) return

    groupRef.current.position.copy(mainCamera.position)

    const currentRotation = new THREE.Euler()
      .setFromQuaternion(mainCamera.quaternion, 'YXZ')
      .y

    groupRef.current.rotation.y = currentRotation

    // 如果有视锥线，也需要在这里同步 fov / rotation
    updateCameraFrustumFromMainCamera(mainCamera)

    mainCamera.userData.isSynchronized = true
  })
}
```

这样右侧主视图拖动时，即使还没写回 store，左上角小地图也能实时更新。

## 15. activeCamera 切换逻辑

左上角点击相机图标时：

```ts
selectObject(cameraId)
setActiveCamera(cameraId)
```

参考：

`src/js/shot-generator/SceneManagerR3fSmall.js:102`

```js
selectObject(match.userData.id)
if(match.userData.type === "camera") {
  setActiveCamera(match.userData.id)
}
```

FlowCanvas 示例：

```tsx
<group
  onPointerDown={(event) => {
    event.stopPropagation()
    selectObject(cameraObject.id)
    setActiveCamera(cameraObject.id)
    beginDrag(event, event.object)
  }}
>
```

状态 reducer / store：

```ts
setActiveCamera: (cameraId) => set({ activeCameraId: cameraId })
```

## 16. 相机距离变化为什么会影响右侧画面

移植后必须保持这个摄影逻辑：

```text
左上角拖动相机靠近人物
→ 改变 camera.x / camera.y
→ 主 PerspectiveCamera 的 position 改变
→ fov 不变
→ 右侧画面出现真实推镜效果
```

这不是 UI 缩放，也不是裁剪，而是透视相机位置变化。

例如：

```text
图 1:
Y = 6.21
Z = 1.09
FOV = 22.3°

图 2:
Y = 2.05
Z = 1.09
FOV = 22.3°
```

变化的是相机与人物距离，不是 FOV。结果：

- 相机靠近人物。
- 人物在画面中变大。
- 构图从全身变成局部。
- 透视关系发生真实变化。

FlowCanvas 需要确保左上角拖动相机时，改的是主相机的世界位置，而不是简单缩放右侧画布。

## 17. 相机参数面板同步

相机面板应该直接读取 active camera state。

```tsx
export function CameraInspector() {
  const activeCamera = useDirectorStore((state) => {
    const id = state.activeCameraId
    if (!id) return null

    const object = state.objects[id]
    return object?.type === 'camera' ? object : null
  })

  const updateObject = useDirectorStore((state) => state.updateObject)

  if (!activeCamera) return null

  return (
    <div>
      <NumberInput
        label="X"
        value={activeCamera.x}
        onChange={(x) => updateObject(activeCamera.id, { x })}
      />

      <NumberInput
        label="Y"
        value={activeCamera.y}
        onChange={(y) => updateObject(activeCamera.id, { y })}
      />

      <NumberInput
        label="Z"
        value={activeCamera.z}
        onChange={(z) => updateObject(activeCamera.id, { z })}
      />

      <NumberInput
        label="旋转"
        value={THREE.MathUtils.radToDeg(activeCamera.rotation)}
        onChange={(rotationDeg) => updateObject(activeCamera.id, {
          rotation: THREE.MathUtils.degToRad(rotationDeg),
        })}
      />

      <NumberInput
        label="倾斜"
        value={THREE.MathUtils.radToDeg(activeCamera.tilt)}
        onChange={(tiltDeg) => updateObject(activeCamera.id, {
          tilt: THREE.MathUtils.degToRad(tiltDeg),
        })}
      />

      <NumberInput
        label="F.O.V."
        value={activeCamera.fov}
        onChange={(fov) => updateObject(activeCamera.id, { fov })}
      />
    </div>
  )
}
```

同步链路：

```text
Inspector 改数值
→ updateObject
→ CameraSync 更新主 Three.js camera
→ 右侧画面变
→ Topdown CameraIcon 根据 cameraObject 更新位置/视锥
```

## 18. 主视图和小视图的同步链路汇总

### A. 左上角拖动相机 → 右侧变化

```text
TopdownViewport pointerdown
→ 命中 CameraIcon
→ selectObject(cameraId)
→ setActiveCamera(cameraId)
→ useTopdownDrag.beginDrag()

TopdownViewport pointermove
→ useTopdownDrag.drag()
→ updateObject(cameraId, { x, y })

Store 更新
→ CameraSync 读取 activeCamera
→ applyCameraStateToThreeCamera()
→ MainViewport PerspectiveCamera 位置变化
→ 右侧主画面实时变化
```

### B. 右侧主场景操作相机 → 左上角变化

```text
MainViewport pointer / keyboard 操作
→ CameraControlsBridge 操作 Three.js PerspectiveCamera
→ camera.userData.isSynchronized = false

Topdown CameraIcon useFrame
→ 读取 mainCanvasRef.current.camera
→ 如果 mainCamera.userData.isSynchronized === false
→ 同步 CameraIcon position / rotation / frustum
→ mainCamera.userData.isSynchronized = true

MainViewport 交互结束
→ threeCameraToCameraState()
→ updateObject(activeCameraId, patch)
→ Store 成为最终可信状态
```

## 19. 必须移植的关键行为清单

### 左上角俯视图

- [ ] 使用 OrthographicCamera。
- [ ] 能显示人物、物体、灯光、图片、相机。
- [ ] 相机显示为图标。
- [ ] 相机图标旁显示名称。
- [ ] 相机图标旁显示焦距/高度信息。
- [ ] 相机图标显示两条视锥线。
- [ ] 点击相机图标切换 active camera。
- [ ] 拖动相机图标只改变 `x / y`，不改变高度 `z`。
- [ ] 拖动相机图标时右侧主画面实时变化。

### 右侧主视图

- [ ] 使用 PerspectiveCamera。
- [ ] 从 active camera state 初始化位置、旋转、fov。
- [ ] 支持 pan / tilt / roll / dolly / move / fov 调整。
- [ ] 操作过程中直接改 Three.js camera，保证流畅。
- [ ] 操作过程中左上角 CameraIcon 实时变化。
- [ ] 操作结束后写回 store。
- [ ] 切换 active camera 时主视图切到对应相机。

### 状态同步

- [ ] 所有对象共享同一份 store。
- [ ] `activeCameraId` 是唯一真实 active camera 标识。
- [ ] `CameraSync` 负责 store → Three.js camera。
- [ ] `CameraControlsBridge` 负责 Three.js camera → store。
- [ ] 小视图 CameraIcon 能通过 mainCanvasRef 读取主 camera。
- [ ] 坐标转换保持一致：`state.y ↔ three.position.z`，`state.z ↔ three.position.y`。

## 20. 常见坑位

### 坑 1：把左上角拖动做成画面缩放

错误做法：

```text
拖动 CameraIcon
→ 改变右侧 canvas scale
```

正确做法：

```text
拖动 CameraIcon
→ 改变真实 PerspectiveCamera position
```

### 坑 2：坐标轴写错

错误：

```ts
camera.position.y = cameraState.y
camera.position.z = cameraState.z
```

正确：

```ts
camera.position.y = cameraState.z
camera.position.z = cameraState.y
```

### 坑 3：右侧拖动过程中只等 pointerup 写状态

如果只在 pointerup 写状态，右侧画面会变，但左上角小图可能不会实时跟随。

必须加：

```ts
camera.userData.isSynchronized = false
```

并让 CameraIcon 在 `useFrame` 中读取主 camera。

### 坑 4：每帧写 store

右侧主视图拖动中不要每帧写 Zustand / Redux，否则容易造成卡顿。

推荐：

```text
拖动中：直接改 Three.js camera
拖动结束：写 store
小视图实时同步：读 mainCamera 引用
```

### 坑 5：FOV 与焦距显示混淆

Three.js 里主参数是 `camera.fov`，UI 可以显示焦距：

```ts
const focalLength = camera.getFocalLength()
```

但内部状态建议仍存 `fov`，不要存焦距，否则转换会复杂。

## 21. 最小可行实现顺序

### Step 1：搭建双 Canvas

- `MainViewport`
- `TopdownViewport`
- 共享 store
- 保存 `mainCanvasRef`

### Step 2：实现 active camera state

- 定义 `DirectorCamera`
- 初始化一个 Camera 1
- `CameraSync` 把状态同步到主 camera

### Step 3：实现左上角 CameraIcon

- 显示位置
- 显示名称
- 显示视锥
- 点击可选中

### Step 4：实现左上角拖动相机

- `useTopdownDrag`
- 拖动写回 `x / y`
- 验证右侧画面实时变化

### Step 5：实现右侧相机控制

- 移植或重写 `CameraControls`
- 拖动中直接改 PerspectiveCamera
- 结束后写回 store

### Step 6：实现右侧 → 左上角实时同步

- CameraIcon 每帧读取 `mainCanvasRef.current.camera`
- 通过 `isSynchronized` 避免重复同步

### Step 7：实现 CameraInspector

- 读取 active camera
- 修改 `x / y / z / rotation / tilt / roll / fov`
- 验证双视图同步

## 22. 参考代码映射表

| 功能 | Storyboarder 文件 | FlowCanvas 建议模块 |
|---|---|---|
| 页面布局 | `Editor/index.js` | `DirectorStage.tsx` |
| 主 3D 场景 | `SceneManagerR3fLarge.js` | `MainViewport.tsx` |
| 左上角俯视图 | `SceneManagerR3fSmall.js` | `TopdownViewport.tsx` |
| 相机图标 | `CameraIcon.js` | `CameraIcon.tsx` |
| 状态同步到主相机 | `CameraUpdate.js` | `CameraSync.tsx` |
| 主场景交互管理 | `InteractionManager.js` | `MainInteractionManager.tsx` |
| 相机控制 | `CameraControlsComponet.js`, `CameraControls.js` | `CameraControlsBridge.tsx`, `DirectorCameraControls.ts` |
| 俯视图拖动 | `use-dragging-manager.js` | `useTopdownDrag.ts` |
| 状态管理 | `shot-generator.js` reducer | `directorStore.ts` |

## 23. 验收用例

### 用例 1：左上角拖动相机靠近人物

操作：

```text
在左上角拖动 Camera 1 靠近人物图标
```

期望：

```text
右侧主场景中人物变大
构图从全身变成半身或局部
FOV 数值不变
CameraInspector 中 Y 值变化
Z 高度不变
```

### 用例 2：左上角拖动相机远离人物

期望：

```text
右侧主场景中人物变小
地面网格露出更多
CameraInspector 中 Y 值变化
```

### 用例 3：右侧主场景旋转相机

期望：

```text
右侧画面旋转
左上角 CameraIcon 方向实时变化
视锥线实时转动
交互结束后 rotation 写回状态
```

### 用例 4：右侧主场景推进相机

期望：

```text
右侧画面出现推镜效果
左上角 CameraIcon 位置实时靠近人物
交互结束后 x/y 写回状态
```

### 用例 5：修改 FOV

期望：

```text
右侧画面视野变宽或变窄
左上角视锥线开合角度变化
相机图标旁焦距显示变化
```

### 用例 6：多相机切换

操作：

```text
左上角点击 Camera 2
```

期望：

```text
activeCameraId 切换为 Camera 2
右侧主场景切到 Camera 2 视角
CameraInspector 显示 Camera 2 参数
Camera 2 在列表中高亮
```

## 24. 结论

这个功能的本质不是“左上角预览图”和“右侧画面”互相截图同步，而是：

```text
两个 Canvas
共享同一份 3D 场景状态
主视图使用真实 PerspectiveCamera
俯视图使用 OrthographicCamera 展示对象布局
左上角拖动相机时写 store
右侧操作相机时先改 Three.js camera，再写 store
左上角 CameraIcon 每帧读取主 camera，实现右侧到左侧的实时同步
```

只要 FlowCanvas 3D 导演台按这个架构实现，就可以 1:1 复刻当前 Shot Generator 的核心导演台体验。
