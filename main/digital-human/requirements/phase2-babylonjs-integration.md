# Phase 2：Babylon.js 3D 模型渲染与集成方案

## 1. 概述

在 Phase 1 桌面应用基础上，引入 **Babylon.js 6.x 3D 渲染引擎**，替代 Phase 1 中的 2D 占位角色展示，实现**3D 数字人角色**的实时渲染、动画驱动和交互。

本文档是**技术方案文档**，重点描述如何将 Babylon.js 3D 场景嵌入 Electron 渲染进程，并与现有音频/网络模块集成。

---

## 2. 为什么选择 Babylon.js

| 对比项 | Babylon.js | Three.js | 结论 |
|--------|-----------|----------|------|
| VRM 支持 | 有官方/社区加载器 | 需自行实现 | Babylon.js 胜 |
| 文档质量 | 极好，playground 丰富 | 好，但分散 | 平手 |
| WebGL 2.0 | 默认启用 | 需手动 | Babylon.js 胜 |
| PBR 材质 | 内置完整管道 | 需配置 | Babylon.js 胜 |
| 粒子系统 | 内置高性能粒子 | 需插件 | Babylon.js 胜 |
| GUI/UI 层 | 内置 GUI | 需额外库 | Babylon.js 胜 |
| 嘴部动画驱动 | MorphTarget/骨骼 API 简洁 | 类似 | 平手 |
| Electron 兼容 | 完全兼容 | 完全兼容 | 平手 |
| 包体积 (tree-shaken) | ~1.2MB | ~800KB | Three.js 略优 |

**核心结论**：Babylon.js 的 **VRM 生态**、**内置粒子** 和 **PBR 材质管线** 使其更适合数字人场景。

---

## 3. 整体架构

```
Phase 1 桌面应用基础设施
         │
         ▼
┌────────────────────────────────────────────────────────┐
│             Electron 主窗口 (对话态)                      │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │           Babylon.js 3D 场景层                │      │
│  │                                              │      │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │      │
│  │  │ 场景管理   │  │ 模型管理  │  │ 动画管理  │   │      │
│  │  │ Scene    │  │ Mesh     │  │ Animati- │   │      │
│  │  │ Engine   │  │ Loading  │  │ onGroup  │   │      │
│  │  │ Camera   │  │ VRM      │  │ Blend-   │   │      │
│  │  │ Lighting │  │ Importer │  │ Shape    │   │      │
│  │  └──────────┘  └──────────┘  └──────────┘   │      │
│  │                                              │      │
│  │  ┌────────────────────────────────────────┐   │      │
│  │  │ 特效层                                   │   │      │
│  │  │ ├── 粒子系统（对话氛围）                  │   │      │
│  │  │ ├── 后处理（Bloom/Glow 光晕）              │   │      │
│  │  │ └── 环境反射探针                          │   │      │
│  │  └────────────────────────────────────────┘   │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │  桥接层 (Babylon ↔ 应用逻辑)                  │      │
│  │                                              │      │
│  │  ├── AudioToMorphBridge: 音频→BlendShape     │      │
│  │  ├── EmotionToExpression: 情绪→表情权重      │      │
│  │  ├── InteractionManager: 点击/拖拽3D交互     │      │
│  │  └── BackgroundRenderer: 背景渲染器          │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │  Phase 1 应用逻辑层（不变）                      │      │
│  │  ├── 音频引擎（复用）                          │      │
│  │  ├── 网络引擎（复用）                          │      │
│  │  └── MCP 工具（复用）                          │      │
│  └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
```

---

## 4. 核心模块设计

### 4.1 SceneManager — 场景管理器

```typescript
class SceneManager {
    // 核心成员
    private engine: BABYLON.Engine;
    private scene: BABYLON.Scene;
    private camera: BABYLON.ArcRotateCamera;
    private light: BABYLON.HemisphericLight;

    // 生命周期
    async initialize(canvas: HTMLCanvasElement): Promise<void>;
    dispose(): void;

    // 场景控制
    startRenderLoop(): void;           // 启动渲染循环
    stopRenderLoop(): void;            // 停止渲染循环
    resize(width: number, height: number): void;

    // 渲染设置
    setBackgroundColor(color: BABYLON.Color4): void;
    enablePostProcess(effect: string): void;  // Bloom, Glow 等
    setEnvironmentTexture(url: string): void; // HDR 环境贴图
}
```

**关键设计决策**：
- 使用 `Engine` 而非 `EngineWithCache`，手动控制渲染循环
- 相机使用 `ArcRotateCamera`（轨道相机），允许用户拖拽旋转视角
- 渲染循环与 `requestAnimationFrame` 整合，与 Electron 窗口 resize 联动
- 场景背景透明（alpha: true），与 Electron 毛玻璃窗口融合

### 4.2 ModelManager — 模型管理器

```typescript
class ModelManager {
    // 核心成员
    private mesh: BABYLON.AbstractMesh | null;
    private skeleton: BABYLON.Skeleton | null;
    private morphTargetManager: BABYLON.MorphTargetManager | null;
    private animations: BABYLON.AnimationGroup[];

    // 模型加载
    async loadModel(modelPath: string): Promise<void>;
    async loadVRM(vrmPath: string): Promise<void>;   // VRM 专用加载器
    unloadModel(): void;

    // 模型属性
    getMorphTargetNames(): string[];           // 获取所有 BlendShape 名称
    getMorphTargetWeight(name: string): number;
    setMorphTargetWeight(name: string, value: number): void;

    // 动画控制
    playAnimation(name: string, loop?: boolean): void;
    stopAnimation(name: string): void;
    blendToAnimation(name: string, duration?: number): void;

    // 模型切换
    async switchModel(modelPath: string): Promise<void>;
}
```

**模型格式支持**：

| 格式 | 加载方式 | BlendShape | 骨骼动画 | 备注 |
|------|---------|-----------|---------|------|
| .glb (标准) | `BABYLON.SceneLoader.ImportMesh` | ✅ | ✅ | 通用格式 |
| .gltf (标准) | 同上 | ✅ | ✅ | 同上 |
| VRM 0.x | 社区 VRM 加载器 | ✅ | ✅ | 日式角色标准 |
| VRM 1.0 | 社区 VRM 加载器（实验性） | ✅ | ✅ | 新版标准 |

### 4.3 MorphTargetBridge — 音频驱动 BlendShape

这是将音频能量转换为 3D 角色口型和表情的核心桥接模块。

```typescript
class MorphTargetBridge {
    private modelManager: ModelManager;
    private analyserNode: AnalyserNode;  // 来自 AudioPlayer

    // 口型映射表（关键配置）
    private visemeMap: VisemeMap = {
        // VRM 标准口型名称 → 中文拼音/音素对应
        'mouthOpen':    ['a', 'o', 'e', 'ā', 'ō', 'ē'],
        'mouthSmile':   ['i', 'ī', 'y'],
        'mouthSad':     ['u', 'ü', 'ū', 'ǜ'],
        'mouthO':       ['ou', 'iu'],
        'mouthAngry':   ['en', 'eng', 'in', 'ing'],
        // ...
    };

    // 更新循环（每帧调用）
    update(deltaTime: number): void {
        // 1. 从 AnalyserNode 获取当前音频能量
        const energy = this.getEnergy();

        // 2. 通过 FFT 分析提取音素特征
        const phoneme = this.analyzePhoneme();

        // 3. 根据音素查找对应的 BlendShape
        const morphTargets = this.visemeMap[phoneme];

        // 4. 平滑过渡到目标权重
        for (const [name, weight] of morphTargets) {
            this.modelManager.setMorphTargetWeight(name, weight);
        }
    }
}
```

**口型驱动策略对比**：

| 方案 | 精度 | 复杂度 | 说明 |
|------|------|--------|------|
| 能量驱动 | 低 | 极低 | 仅根据音频能量大小控制 mouthOpen 权重，不区分音素 |
| 频段驱动 | 中 | 低 | 将 FFT 频段映射到多个 BlendShape（低频→O型，高频→I型） |
| 音素映射 | 高 | 中 | 实时分析音频音素，映射到 VRM 标准口型 | 
| 文本驱动 | 最高 | 高 | 从 LLM 返回文本提取音素，直接驱动口型 |

**Phase 2 推荐采用"频段驱动"方案**，复杂度低、效果可接受。后续可升级到音素映射方案。

### 4.4 EmotionManager — 情绪表情管理

接收 WebSocket 消息中的 `emotion` 字段，驱动 3D 模型的 BlendShape 表情。

```typescript
class EmotionManager {
    // 情绪 → BlendShape 权重映射
    private emotionMap: EmotionMap = {
        'happy':     { mouthSmile: 0.8, eyeOpen: 0.5, cheekPuff: 0.3 },
        'sad':       { mouthSad: 0.7, eyeClosed: 0.3, browDown: 0.6 },
        'angry':     { mouthAngry: 0.6, browDown: 0.8, eyeOpen: 0.4 },
        'surprised': { mouthOpen: 0.7, eyeOpen: 1.0, browUp: 0.9 },
        'neutral':   { /* 默认态 */ },
        // ...
    };

    // 表情切换（带平滑过渡）
    transitionToEmotion(emotion: string, duration: number = 0.3): void {
        // 线性插值当前权重到目标权重
    }

    // 自动眨眼（定时任务，与主表情叠加）
    autoBlink(interval: number = 4000): void;    // 每 4s 眨眼一次
    // 自动呼吸（身体微动）
    autoBreathe(): void;    // 胸部轻微起伏
    // 眼球自动运动（随机注视方向）
    autoEyeMovement(): void;
}
```

### 4.5 InteractionManager — 3D 交互管理器

```typescript
class InteractionManager {
    private scene: BABYLON.Scene;

    // 启用交互
    enable(canvas: HTMLCanvasElement): void;

    // 单击检测（使用 Raycaster）
    onSingleClick(mesh: BABYLON.AbstractMesh): void;

    // 双击检测
    onDoubleClick(mesh: BABYLON.AbstractMesh): void;

    // 拖拽旋转视角
    enableOrbitControl(): void;

    // 触摸/手势支持
    enableTouchGestures(): void;
}
```

**交互行为**：

| 操作 | 效果 |
|------|------|
| 鼠标拖拽 | 旋转视角（围绕角色 ArcRotateCamera） |
| 滚轮 | 拉近/拉远视角 |
| 单击角色头部 | 触发问候动画 |
| 单击角色身体 | 触发互动动画 |
| 双击角色 | 触发特殊动作（跳舞/转圈等） |
| 拖拽背景空白 | 无操作（与窗口拖拽不冲突） |

---

## 5. VRM 模型标准与资源

### 5.1 VRM 模型 BlendShape 标准名称

VRM 0.x 标准预定义的 BlendShape 名称：

| 预设 | BlendShape 名称 | 用途 |
|------|----------------|------|
| A | `mouthOpen` | 张口（啊音） |
| I | `mouthSmile` | 微笑（依音） |
| U | `mouthSad` | 嘟嘴（呜音） |
| E | `mouthAngry` | 咧嘴（诶音） |
| O | `mouthO` | 圆口（哦音） |
| 眨眼 | `eyeClosed` / `eyeBlinkLeft` / `eyeBlinkRight` | 闭眼 |
| 高兴 | `mouthSmile` + `eyeOpen` | 笑容 |
| 悲伤 | `mouthSad` + `eyeClosed` | 悲伤表情 |
| 生气 | `mouthAngry` + `browDown` | 愤怒表情 |
| 惊讶 | `mouthOpen` + `eyeOpen` | 惊讶表情 |
| 无表情 | （默认姿态） | 中性表情 |

### 5.2 VRM 模型获取渠道

| 渠道 | 费用 | 质量 | 商用许可 |
|------|------|------|---------|
| VRoid Hub | 免费/付费 | 高 | 需确认每个模型许可 |
| Booth.pm | ¥500~5000 | 极高 | 通常可商用 |
| Sketchfab | 免费/付费 | 中-高 | 需确认许可 |
| 自行制作 (VRoid Studio) | 免费工具 | 高 | 完全自主 |
| 自行制作 (Blender) | 免费 | 自定义 | 完全自主 |

### 5.3 模型加载示例代码

```typescript
async function loadVRMModel(
    scene: BABYLON.Scene,
    vrmPath: string
): Promise<BABYLON.AbstractMesh> {
    return new Promise((resolve, reject) => {
        BABYLON.SceneLoader.ImportMesh(
            '',                                         // mesh 名称
            '',                                         // 根路径
            vrmPath,                                    // 文件名
            scene,
            (meshes, particleSystems, skeletons) => {
                const rootMesh = meshes[0];
                // 自动缩放和定位
                rootMesh.scaling = new BABYLON.Vector3(1.5, 1.5, 1.5);
                rootMesh.position = new BABYLON.Vector3(0, -1.2, 0);
                resolve(rootMesh);
            },
            null,
            (scene, message, exception) => {
                reject(new Error(`VRM 加载失败: ${message}`));
            },
            '.glb'
        );
    });
}
```

---

## 6. 特效系统

### 6.1 粒子系统（对话氛围）

```typescript
class DialogueParticles {
    private particleSystem: BABYLON.ParticleSystem;

    constructor(scene: BABYLON.Scene) {
        this.particleSystem = new BABYLON.ParticleSystem(
            'dialogueParticles', 200, scene
        );
        this.particleSystem.particleTexture = new BABYLON.Texture(
            'assets/particle.png', scene
        );
        this.particleSystem.emitter = new BABYLON.Vector3(0, 1, 0);
        this.particleSystem.minSize = 0.02;
        this.particleSystem.maxSize = 0.05;
        this.particleSystem.minLifeTime = 1.0;
        this.particleSystem.maxLifeTime = 3.0;
        this.particleSystem.minEmitPower = 0.1;
        this.particleSystem.maxEmitPower = 0.3;
        this.particleSystem.color1 = new BABYLON.Color4(0.7, 0.8, 1.0, 1.0);
        this.particleSystem.color2 = new BABYLON.Color4(0.9, 0.9, 1.0, 0.8);
        this.particleSystem.colorDead = new BABYLON.Color4(0, 0, 0, 0);
    }

    start(): void;
    stop(): void;
    setIntensity(level: number): void;   // 根据音频能量调整粒子密度
}
```

### 6.2 后处理特效

```typescript
// Bloom（泛光）效果 — 对话时角色周围发光
const bloom = new BABYLON.BloomEffect(
    scene, 0.5,     // 亮度阈值
    0.8,            // 混合系数
    0.3,            // 模糊程度
    1.0,            // 权重
    BABYLON.Texture.BILINEAR_SAMPLINGMODE
);

// 待机时关闭 Bloom，对话时开启
function setDialogueMode(active: boolean) {
    bloom.isEnabled = active;
}
```

### 6.3 背景渲染方案

| 方案 | 实现 | 适用场景 |
|------|------|---------|
| 纯色渐变 | 用场景 clearColor | 简约风格 |
| HDR 环境贴图 | 加载 .hdr 文件 | 写实风格（配合 PBR 材质） |
| 动态渐变 | 每帧更新 clearColor | 情绪氛围 |
| 毛玻璃效果 | Babylon.js 场景透明 + 窗口 backdrop-filter | 与桌面融合 |
| 星空粒子背景 | 大量散布粒子 | 科技感 |

**推荐方案**：场景透明 + Electron 窗口的 `backdrop-filter: blur(20px)` 毛玻璃效果。角色周围增加 Bloom 特效增强立体感。

---

## 7. 性能优化策略

### 7.1 渲染优化

| 优化项 | 方法 | 预期收益 |
|--------|------|---------|
| 多边形数量控制 | VRM 模型控制在 20K 面以内 | 基础 |
| 纹理大小 | 基础纹理 ≤ 2048px，法线贴图 ≤ 1024px | 显存减半 |
| LOD | LOD 层级（远距离用低面模型） | 可选 |
| 不必要骨骼剔除 | 禁用不影响外观的 SpringBone | CPU 负载降低 |
| 粒子数量限制 | 对话粒子 ≤ 500 个 | 稳定 60fps |
| 后处理按需开关 | 沉默时关闭 Bloom | GPU 负载减半 |

### 7.2 Electron 侧优化

- 使用 `WebGL 2.0` 而非 WebGL 1.0
- 渲染进程启用 `gpu` 沙箱
- 窗口非激活时降低帧率（`engine.render()` 改为 30fps）
- 使用 `OffscreenCanvas` 渲染到背景线程？不支持，但可以使用 Worker

### 7.3 帧率管理策略

```typescript
class AdaptiveFrameRate {
    private targetFPS: number = 60;
    private lastFrameTime: number = 0;

    // 根据窗口可见性和对话状态调整帧率
    update(windowVisible: boolean, isTalking: boolean): void {
        if (windowVisible && isTalking) {
            this.targetFPS = 60;  // 对话中全帧率
        } else if (windowVisible && !isTalking) {
            this.targetFPS = 30;  // 待机半帧率
        } else {
            this.targetFPS = 15;  // 后台最低帧率
        }
    }
}
```

---

## 8. 小圆窗 3D 化升级（可选）

Phase 2 中，小圆窗（待机态）也可以从 2D Canvas 升级为 **微型 3D 场景**：

```
小型 3D 场景（60×60px 渲染）：
    ┌──────────────┐
    │   ● 3D 角色   │  ← 使用低面模型 + 简化渲染
    │   微型化      │
    └──────────────┘
```

**方案对比**：

| 方案 | 说明 | 复杂度 | GPU 消耗 |
|------|------|--------|---------|
| 保留 2D Canvas | Phase 1 的呼吸光晕 | 低 | 几乎为零 |
| 微型 3D 渲染 | 另开一个 60×60 的 WebGL 上下文 | 中 | 较高 |
| 主窗口缩略截图 | 主窗口截取一小块显示 | 中 | 需另开子窗口 |

**推荐**：Phase 2 保留 2D Canvas 小圆窗，3D 渲染只在主窗口进行。性能开销和复杂度可控。

---

## 9. 与 Phase 1 的集成点

### 9.1 HTML 模板变化

```html
<!-- Phase 1: 2D Canvas 角色 -->
<div id="character-area">
    <canvas id="character-canvas"></canvas>
</div>

<!-- Phase 2: Babylon.js 3D 场景替换 -->
<div id="babylon-container">
    <canvas id="babylon-canvas"></canvas>
</div>
```

### 9.2 初始化流程变化

```typescript
// Phase 1 初始化
class MainApp {
    async init() {
        this.audioEngine.init();
        this.networkEngine.init();
        this.characterDisplay = new Character2D('character-canvas');
    }
}

// Phase 2 初始化
class MainApp {
    async init() {
        this.audioEngine.init();
        this.networkEngine.init();

        // 3D 场景初始化
        this.sceneManager = new SceneManager();
        await this.sceneManager.initialize('babylon-canvas');

        this.modelManager = new ModelManager();
        await this.modelManager.loadModel('assets/model/vrm/model.glb');

        this.morphBridge = new MorphTargetBridge(
            this.modelManager,
            this.audioEngine.analyser
        );

        this.emotionManager = new EmotionManager(this.modelManager);
        this.interactionManager = new InteractionManager(this.sceneManager.scene);

        // 连接网络事件
        this.networkEngine.onEmotionChange = (emotion) => {
            this.emotionManager.transitionToEmotion(emotion);
        };
    }
}
```

### 9.3 与现有音频引擎的对接

```typescript
// Phase 1 音频引擎 (复用)
class AudioPlayer {
    getAnalyser(): AnalyserNode;  // 已存在

    // 新增：注册 MorphTargetBridge 的更新回调
    onAudioFrame: ((frequencyData: Uint8Array) => void) | null;
}

// 在渲染循环中调用 MorphTargetBridge.update
sceneManager.onBeforeRender = () => {
    const analyser = audioPlayer.getAnalyser();
    const data = new Uint8Array(analyser.frequencyBinCount);
    analyser.getByteFrequencyData(data);
    morphBridge.update(data);
};
```

---

## 10. 项目目录变化（与 Phase 1 对比）

```
xiaozhi-desktop/
├── ... (Phase 1 所有文件保持不变)
│
└── src/renderer/
    ├── scripts/
    │   ├── bubble-app.js          ← Phase 1 不变
    │   ├── main-app.js            ← Phase 1 修改：初始化 3D 场景
    │   └── babylon/
    │       ├── index.ts                # Babylon.js 入口
    │       ├── SceneManager.ts         # 场景管理器
    │       ├── ModelManager.ts         # 模型加载器
    │       ├── MorphTargetBridge.ts    # 音频→BlendShape 桥接
    │       ├── EmotionManager.ts       # 情绪表情管理
    │       ├── InteractionManager.ts   # 3D 交互管理
    │       ├── DialogueParticles.ts    # 对话粒子系统
    │       ├── EffectsManager.ts       # 后处理特效管理
    │       └── viseme-map.ts           # 口型映射配置
    ├── assets/
    │   └── models/                # 3D 模型文件
    │       ├── vrm/               # VRM 模型目录
    │       └── env/               # HDR 环境贴图
    └── styles/
        └── main.css               ← 修改：适配 3D 场景布局
```

**包依赖新增**：

```json
{
  "dependencies": {
    "@babylonjs/core": "^6.0.0",
    "@babylonjs/materials": "^6.0.0",
    "@babylonjs/loaders": "^6.0.0",
    "@babylonjs/post-processes": "^6.0.0",
    "@babylonjs/procedural-textures": "^6.0.0"
  }
}
```

TypeScript 配置：

```json
{
  "compilerOptions": {
    "strict": true,
    "moduleResolution": "node",
    "esModuleInterop": true,
    "target": "ES2020",
    "module": "ESNext",
    "outDir": "dist/renderer/scripts/babylon"
  }
}
```

---

## 11. 模型准备流程

### 11.1 获取模型的完整流程

```
1. 获取 VRM 模型
   ├── VRoid Studio (免费, 自己捏人)
   ├── VRoid Hub 下载 (免费/付费)
   └── Booth.pm 购买 (付费)

2. 模型检查与优化
   ├── 使用 VRM 查看器确认 BlendShape 名称
   ├── 确认口型 BlendShape 存在 (mouthOpen, mouthSmile 等)
   ├── 确认表情 BlendShape 存在 (eyeOpen, browDown 等)
   └── 确认骨骼合规 (head, neck, spine 等)

3. 导出为 .glb 格式
   ├── Unity + UniVRM → 导出 .vrm → 解压为 .glb
   └── Blender + VRM Add-on → 导出 .glb

4. 放入项目 assets/models/ 目录
```

### 11.2 模型文件打包策略

```yaml
# electron-builder.yml 附加配置
extraResources:
  - from: assets/models
    to: models
    filter:
      - "**/*.glb"
      - "**/*.gltf"
      - "**/*.bin"
      - "!**/*.blend"       # 排除源文件
      - "!**/*.psd"
      - "!**/*.zip"
```

---

## 12. 风险与应对

| 风险 | 概率 | 影响 | 应对 |
|------|------|------|------|
| VRM 模型 BlendShape 命名不标准 | 中 | 高 | 开发 VisemeMap 配置器，允许用户手动映射 |
| WebGL 上下文丢失（切窗口/休眠） | 低 | 高 | 监听 `contextlost` 事件，自动重建场景 |
| 模型加载慢（首次 3-5s） | 低 | 中 | 加载界面显示进度条；考虑预加载 |
| 性能不足（集成显卡 60fps 不达标） | 低 | 中 | 自动检测 GPU 级别，提供画质选项（低/中/高） |
| Electron 多窗口增加内存 | 低 | 低 | Phase 2 维持单窗口策略，小圆窗独立但无 3D 渲染 |
| Babylon.js 版本升级断裂 | 低 | 中 | 锁定次要版本号，升级前测试 |

---

## 13. Phase 2 阶段排期

| 阶段 | 内容 | 工时 |
|------|------|------|
| M1 | Babylon.js 场景搭建 + 模型加载 | 2 天 |
| M2 | 音频→BlendShape 桥接（口型驱动） | 2 天 |
| M3 | 情绪表情 + 自动眨眼/呼吸/眼动 | 1.5 天 |
| M4 | 粒子特效 + 后处理（Bloom等） | 1 天 |
| M5 | 3D 交互（点击/拖拽视角） | 1 天 |
| M6 | 性能调优 + 降级策略 | 1 天 |
| M7 | 模型准备工具 + 文档 | 0.5 天 |

**总计：约 9 天**（在 Phase 1 完成的基础上）

---

## 14. 整体路线图

```
Phase 1（桌面圆形UI + 核心通信）
     │
     │  ╔══ Phase 1 交付 ═══════════════════╗
     │  ║  - Electron 桌面应用               ║
     │  ║  - 圆形呼吸光晕小圆窗              ║
     │  ║  - 主窗口 UI（2D 角色占位）         ║
     │  ║  - 音频/网络/Wakeword 完整链路      ║
     │  ╚═══════════════════════════════════╝
     │
     ▼
Phase 2（3D 模型渲染 + 特效）
     │
     │  ╔══ Phase 2 交付 ═══════════════════╗
     │  ║  - Babylon.js 3D 场景             ║
     │  ║  - VRM/.glb 模型加载              ║
     │  ║  - 音频驱动口型动画                ║
     │  ║  - 情绪表情系统                    ║
     │  ║  - 粒子特效 + 对话氛围             ║
     │  ╚═══════════════════════════════════╝
     │
     ▼
Phase 3（未来可扩展方向，非本文档范围）
     ├── 多模型选择/换装
     ├── 背景场景换装
     ├── 屏幕录制/截图分享
     ├── 语音唤醒词联邦学习
     ├── 本地 LLM 集成
     └── 蓝牙/串口硬件联动（实体数字人）
```