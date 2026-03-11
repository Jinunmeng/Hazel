# Hazel 引擎架构分析

## 项目概述

Hazel 是一个基于 C++ 的早期阶段游戏引擎，主要面向 Windows 平台，同时也是一个教学工具，用于讲解游戏引擎的设计与架构。项目由 TheCherno（Yan Chernikov）在 YouTube 系列视频中开发。

---

## 顶层项目结构

```
Hazel/
├── Hazel/               # 核心引擎静态库
├── Hazel-ScriptCore/    # C# 脚本核心库（.NET 程序集）
├── Hazelnut/            # 编辑器应用（可执行文件）
├── Sandbox/             # 测试/示例应用（可执行文件）
├── vendor/              # 第三方依赖
├── scripts/             # 构建/安装脚本
├── Resources/           # 引擎资源（logo 等）
└── premake5.lua         # 构建系统配置（premake5）
```

项目使用 **premake5** 作为构建系统，支持 Debug、Release、Dist 三种配置，面向 x86_64 架构。

---

## 核心引擎模块（`Hazel/src/Hazel/`）

### 1. Core — 核心基础设施

| 文件 | 职责 |
|------|------|
| `Application` | 应用程序主类，管理主循环、窗口、层栈、事件分发 |
| `Window` | 平台无关的窗口抽象（接口） |
| `Layer / LayerStack` | 分层架构，每帧按序更新；Overlay 层叠加在最上方 |
| `Input` | 平台无关输入查询接口（键盘/鼠标） |
| `Log` | 日志系统（基于 spdlog），区分 Core 日志与 Client 日志 |
| `UUID` | 唯一标识符，用于 Entity 身份标识 |
| `Timestep` | 帧间隔时间封装（delta time） |
| `Timer` | 高精度计时器 |
| `Buffer` | 通用原始内存缓冲区 |
| `FileSystem` | 文件系统工具函数 |
| `EntryPoint` | 定义 `main` 函数入口，由客户端实现 `CreateApplication()` |
| `Assert / Base` | 断言宏及引擎基础类型定义（`Ref<T>`, `Scope<T>` 等） |

**应用启动流程：**
```
main() [EntryPoint.h]
  └─> CreateApplication()  [由客户端实现]
        └─> Application::Run()
              └─> 主循环: LayerStack::OnUpdate() → ImGui渲染 → Window::OnUpdate()
```

---

### 2. Events — 事件系统

事件系统采用**同步分发**模式，通过模板化的 `EventDispatcher` 将事件路由到对应处理函数。

```
Event（基类）
├── ApplicationEvent
│   ├── WindowCloseEvent
│   ├── WindowResizeEvent
│   └── AppTickEvent / AppUpdateEvent / AppRenderEvent
├── KeyEvent
│   ├── KeyPressedEvent
│   ├── KeyReleasedEvent
│   └── KeyTypedEvent
└── MouseEvent
    ├── MouseMovedEvent
    ├── MouseScrolledEvent
    ├── MouseButtonPressedEvent
    └── MouseButtonReleasedEvent
```

---

### 3. Renderer — 渲染系统

渲染系统采用**抽象层 + 具体实现**的分层设计，当前仅实现 OpenGL 后端。

```
渲染抽象层（Hazel/Renderer/）
├── RendererAPI        ← 纯虚接口：Clear, DrawIndexed, SetViewport 等
├── RenderCommand      ← 静态封装 RendererAPI，供上层调用
├── Renderer           ← 3D 场景渲染（BeginScene / EndScene / Submit）
├── Renderer2D         ← 高性能 2D 批量渲染（Quads、Circles、Lines、Text）
├── Buffer             ← VertexBuffer / IndexBuffer 抽象
├── VertexArray        ← VAO 抽象
├── Shader             ← GLSL Shader 抽象（支持文本/二进制）
├── Texture            ← 2D 纹理抽象
├── Framebuffer        ← 离屏渲染帧缓冲（含 EntityID 附件用于像素拾取）
├── UniformBuffer      ← Uniform Buffer Object 抽象
├── GraphicsContext    ← 图形上下文抽象（OpenGL Context）
├── Camera             ← 通用相机基类（projection matrix）
├── OrthographicCamera ← 2D 正交相机
├── EditorCamera       ← 编辑器自由视角相机
├── OrthographicCameraController ← 正交相机控制器
├── Font               ← MSDF 字体渲染
└── MSDFData           ← MSDF 字体内部数据

OpenGL 实现（Platform/OpenGL/）
├── OpenGLRendererAPI
├── OpenGLBuffer
├── OpenGLVertexArray
├── OpenGLShader
├── OpenGLTexture
├── OpenGLFramebuffer
├── OpenGLUniformBuffer
└── OpenGLContext
```

**Renderer2D 批量渲染机制：**  
每帧收集所有 Quad/Circle/Line/Text 绘制调用，合并为单次 DrawCall，支持自动 Flush（超出批次容量时）。

---

### 4. Scene — 场景与实体组件系统（ECS）

场景系统基于第三方库 **entt** 实现 ECS 架构。

```
Scene
├── entt::registry      ← ECS 注册表，管理所有 Entity 和 Component
├── Entity              ← entt::entity 的 C++ 包装，提供组件操作接口
├── Components.h        ← 所有组件定义（纯数据结构）
│   ├── IDComponent         (UUID)
│   ├── TagComponent        (name string)
│   ├── TransformComponent  (Translation / Rotation / Scale)
│   ├── CameraComponent     (SceneCamera + Primary flag)
│   ├── SpriteRendererComponent  (Color + Texture + TilingFactor)
│   ├── CircleRendererComponent  (Color + Thickness + Fade)
│   ├── TextComponent       (TextString + Font + Color + Kerning)
│   ├── ScriptComponent     (C# 类名)
│   ├── NativeScriptComponent   (C++ 原生脚本)
│   ├── Rigidbody2DComponent    (物理刚体类型)
│   ├── BoxCollider2DComponent  (盒形碰撞体)
│   └── CircleCollider2DComponent (圆形碰撞体)
├── ScriptableEntity    ← C++ 原生脚本基类
├── SceneCamera         ← 场景相机（支持透视/正交投影）
└── SceneSerializer     ← 场景序列化/反序列化（YAML 格式）
```

**场景生命周期：**
```
OnRuntimeStart()    ← 启动物理世界、初始化脚本
OnUpdateRuntime()   ← 每帧：执行脚本 → 更新物理 → 渲染
OnRuntimeStop()     ← 停止物理、清理脚本

OnSimulationStart() ← 仅启动物理（无脚本）
OnUpdateSimulation()← 每帧：更新物理 → 渲染

OnUpdateEditor()    ← 编辑模式：仅渲染（使用 EditorCamera）
```

---

### 5. Scripting — C# 脚本系统

脚本系统通过 **Mono** 运行时嵌入 C# 支持。

```
ScriptEngine
├── InitMono()          ← 初始化 Mono 运行时
├── LoadAssembly()      ← 加载 Hazel-ScriptCore.dll（引擎 C# 核心库）
├── LoadAppAssembly()   ← 加载用户项目的 C# 程序集
├── ReloadAssembly()    ← 热重载 C# 程序集
├── OnCreateEntity()    ← 调用 C# Entity.OnCreate()
├── OnUpdateEntity()    ← 调用 C# Entity.OnUpdate(ts)
├── ScriptClass         ← 映射到 Mono 中的 C# 类
├── ScriptInstance      ← C# 类实例，持有 MonoObject*
└── ScriptGlue          ← C++ ↔ C# 绑定（Internal Calls）

Hazel-ScriptCore/      ← C# 脚本 API
  └── Entity / Component / Input / ... 等供用户继承和调用的 C# 类
```

---

### 6. Physics — 物理系统

物理系统封装了 **Box2D** 2D 物理引擎。  
`Scene` 持有 `b2World*`，在 `OnRuntimeStart()` / `OnSimulationStart()` 时创建物理世界，遍历带有 `Rigidbody2DComponent` 的 Entity 创建对应 Box2D Body，并在每帧 `Step()` 后将物理结果同步回 `TransformComponent`。

---

### 7. ImGui — 调试 UI

基于 **Dear ImGui** 实现调试界面层。  
`ImGuiLayer` 继承自 `Layer`，在 `OnImGuiRender()` 中绘制 ImGui 内容。使用 GLFW + OpenGL 作为 ImGui 后端。

---

### 8. Project — 项目管理

```
Project
├── ProjectConfig  ← 项目名称、资源目录、脚本目录等配置
├── Project        ← 单例，管理当前活跃项目
└── ProjectSerializer ← 项目文件序列化（YAML 格式，.hproj 文件）
```

---

## 平台层（`Hazel/src/Platform/`）

| 目录 | 实现内容 |
|------|----------|
| `Platform/OpenGL/` | OpenGL 图形后端：Buffer、Context、Framebuffer、RendererAPI、Shader、Texture、UniformBuffer、VertexArray |
| `Platform/Windows/` | Windows 平台：窗口（基于 GLFW）、输入（基于 GLFW）、平台工具函数（文件对话框） |

---

## 编辑器（Hazelnut）

Hazelnut 是基于 Hazel 引擎构建的场景编辑器，其主体为 `EditorLayer`（继承自 `Layer`）。

```
EditorLayer
├── 视口 (Viewport)         ← 渲染到 Framebuffer，在 ImGui 窗口中显示
├── SceneHierarchyPanel     ← 场景实体层级树 + Entity 属性面板
├── ContentBrowserPanel     ← 项目资产浏览器（文件系统导航）
├── EditorCamera            ← 编辑器自由视角相机
├── ImGuizmo                ← 3D 变换 Gizmo（位移/旋转/缩放）
└── SceneState              ← Edit / Play / Simulate 三种模式切换
```

**编辑器工作流：**
```
Edit 模式 → OnScenePlay() → Play 模式（运行脚本+物理）
Edit 模式 → OnSceneSimulate() → Simulate 模式（仅运行物理）
Play/Simulate 模式 → OnSceneStop() → 返回 Edit 模式
```

---

## 测试应用（Sandbox）

Sandbox 是演示和测试引擎功能的应用程序，包含 `ExampleLayer`（3D 渲染示例）和 `Sandbox2D`（2D 批量渲染示例）。

---

## 第三方依赖总览

| 库 | 用途 |
|----|------|
| **GLFW** | 跨平台窗口创建与输入处理 |
| **Glad** | OpenGL 函数加载器 |
| **Dear ImGui** | 即时模式 GUI，用于编辑器 UI |
| **ImGuizmo** | ImGui 内的 3D 变换 Gizmo |
| **glm** | OpenGL 数学库（向量、矩阵、四元数） |
| **entt** | 高性能 Entity-Component-System 库 |
| **Box2D** | 2D 物理引擎 |
| **yaml-cpp** | YAML 序列化（场景、项目文件存储） |
| **msdf-atlas-gen** | 多通道有符号距离场字体图集生成 |
| **stb_image** | 图像文件加载（PNG/JPG 等） |
| **Mono** | C# 脚本运行时 |
| **spdlog** | 高性能 C++ 日志库 |

---

## 架构关键设计模式

| 模式 | 应用场景 |
|------|----------|
| **单例（Singleton）** | `Application`、`ScriptEngine`、`Project` |
| **工厂方法（Factory Method）** | `RendererAPI::Create()`、`Texture2D::Create()` 等，根据平台创建具体实现 |
| **分层架构（Layered Architecture）** | `LayerStack` 管理渲染层、UI层、业务逻辑层 |
| **ECS（Entity-Component-System）** | 场景中的所有游戏对象通过 entt 管理 |
| **观察者/事件（Observer/Event）** | 事件系统通过 `EventDispatcher` 分发窗口、键盘、鼠标事件 |
| **命令（Command）** | `RenderCommand` 封装渲染 API 调用 |
| **代理/桥接（Bridge）** | 渲染抽象层（如 `Shader`）与具体实现（`OpenGLShader`）分离 |

---

## 模块依赖关系图

```
           ┌──────────────────────────────────────┐
           │           Hazelnut (Editor)           │
           │   EditorLayer / Panels / ImGuizmo     │
           └───────────────────┬──────────────────┘
                               │ uses
           ┌───────────────────▼──────────────────┐
           │         Hazel Engine Library          │
           │                                       │
           │  Core ─── Events ─── ImGui            │
           │   │                                   │
           │  Renderer ─── Scene ─── Scripting     │
           │   │              │                    │
           │  Platform    Physics (Box2D)           │
           │  (OpenGL/Win)                         │
           └───────────────────────────────────────┘
                               │ scripting bridge
           ┌───────────────────▼──────────────────┐
           │       Hazel-ScriptCore (C# .NET)      │
           └───────────────────────────────────────┘
```
