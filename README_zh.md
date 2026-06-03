# TauriCPP

基于 WebView2 的轻量级 C++ 桌面应用框架，灵感来自 [Tauri](https://tauri.app)。使用 Web 前端 + C++ 后端构建现代桌面应用——单文件部署，源码不泄露。

## 特性

- **单 EXE 部署** — WebView2Loader.dll 和所有前端资源均嵌入可执行文件，无需任何外部文件。
- **源码保护** — 前端资源（HTML/CSS/JS）编译时作为 Windows 资源嵌入 exe，运行时通过 VirtualFS 从内存提供，不生成临时文件。
- **双向通信** — JS 调用 C++ 命令（Promise 风格），C++ 向 JS 推送事件。
- **开发/生产双模式** — 开发时自动从文件系统加载，生产时从嵌入资源加载，无需修改代码。
- **极简依赖** — 仅需 WebView2 SDK 和 nlohmann/json，不捆绑 Chromium，不依赖 .NET 运行时，不依赖 Qt。
- **C++17** — 现代 C++，API 简洁清晰。

## 快速开始

### 环境要求

- Visual Studio 2019+（需安装 C++ 桌面开发工作负载）
- [vcpkg](https://vcpkg.io)（classic 模式，安装在 `C:\vcpkg`）
- Python 3.7+（用于资源打包脚本）
- PowerShell 5.1+

### 安装依赖（首次使用）

```powershell
.\build.ps1 -SetupDeps
```

此命令将：
1. 下载 `nlohmann/json.hpp` 到 `third_party/` 目录
2. 通过 vcpkg 安装 `webview2`（`vcpkg install webview2:x64-windows`）

### 构建

```powershell
.\build.ps1              # 构建（Release 配置）
.\build.ps1 -Clean       # 清理构建目录并重新构建
```

输出：`build\sample.exe`

### 运行

```powershell
.\build\sample.exe
```

## 项目结构

```
TauriCPP/
├── CMakeLists.txt              # CMake 构建配置
├── build.ps1                   # PowerShell 构建脚本（Ninja + MSVC）
├── vcpkg.json                  # vcpkg 依赖清单
├── include/tauricpp/
│   ├── app.hpp                 # 应用入口
│   ├── bridge.hpp              # 前后端通信桥
│   ├── window.hpp              # Win32 + WebView2 窗口
│   ├── virtual_fs.hpp          # 内存虚拟文件系统
│   └── embedded_dll.hpp        # 嵌入式 DLL 加载器（delay-load 钩子）
├── src/
│   ├── app.cpp
│   ├── bridge.cpp              # 含注入前端的 JS 桥接代码
│   ├── window.cpp              # WebView2 初始化与资源拦截
│   ├── virtual_fs.cpp
│   └── embedded_dll.cpp        # delay-load 钩子（__pfnDliNotifyHook2）
├── sample/
│   ├── src/main.cpp            # 示例应用
│   └── frontend/
│       ├── index.html
│       ├── css/style.css
│       └── js/app.js
├── tools/
│   └── pack_resources.py       # 前端文件 -> .rc 资源打包工具
└── third_party/
    └── nlohmann/json.hpp       # JSON 库（单头文件）
```

## 使用方法

### 1. 创建应用

```cpp
#include "tauricpp/app.hpp"
#include "tauricpp/embedded_dll.hpp"

int WINAPI WinMain(HINSTANCE, HINSTANCE, LPSTR, int) {
    // 从 exe 资源段加载嵌入的 WebView2Loader.dll
    tauricpp::EmbeddedDll::Load("1", "EMBEDDED_DLL", "WebView2Loader.dll");

    // 配置应用
    tauricpp::App::Config config;
    config.window_config.title = "我的应用";
    config.window_config.width = 1200;
    config.window_config.height = 800;
    config.window_config.center = true;

    tauricpp::App app(config);

    // 注册前端可调用的命令
    app.GetBridge().RegisterCommand("greet", [](const nlohmann::json& args) {
        std::string name = args.value("name", "World");
        return nlohmann::json{{"message", "你好, " + name + "!"}};
    });

    // 向前端推送事件
    app.GetBridge().Emit("timer", nlohmann::json{{"tick", 1}});

    return app.Run();
}
```

### 2. 前端 JavaScript API

```javascript
// JS -> C++：调用命令（返回 Promise）
const result = await __tauricpp__.invoke('greet', { name: 'World' });
// result: { "message": "你好, World!" }

// C++ -> JS：监听事件
__tauricpp__.listen('timer', function(data) {
    console.log('定时器:', data);
});

// 移除监听
__tauricpp__.removeListener('timer', callback);
```

### 3. 将前端打包进 EXE

`sample/frontend/` 目录中的前端文件在构建时由 `tools/pack_resources.py` 自动打包为 Windows 资源脚本并编译进 exe。运行时，`VirtualFS` 从 exe 资源段加载文件，通过 `WebResourceRequested` 拦截请求从内存提供内容——不写入任何临时文件到磁盘。

## 架构

```
┌─────────────────────────────────────────────────┐
│                   sample.exe                     │
│                                                  │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐ │
│  │ App      │  │ Bridge    │  │ Window       │ │
│  │          │──│           │──│              │ │
│  │ Config   │  │ Register  │  │ Win32 +      │ │
│  │ Run()    │  │ Emit()    │  │ WebView2     │ │
│  └──────────┘  └───────────┘  └──────┬───────┘ │
│                                       │         │
│  ┌──────────────┐  ┌─────────────────┐│         │
│  │ VirtualFS    │  │ EmbeddedDll     ││         │
│  │              │  │                 ││         │
│  │ RegisterFile │  │ 从资源段加载     ││         │
│  │ FindFile     │  │ delay-load 钩子 ││         │
│  └──────┬───────┘  └─────────────────┘│         │
│         │                              │         │
│  ┌──────▼──────────────────────────────▼───────┐ │
│  │         Windows 资源段                       │ │
│  │  ┌─────────────┐  ┌──────────────────────┐  │ │
│  │  │ WebView2    │  │ 前端资源              │  │ │
│  │  │ Loader.dll  │  │ (HTML/CSS/JS/...)     │  │ │
│  │  └─────────────┘  └──────────────────────┘  │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 运行机制

1. **构建时**：`pack_resources.py` 扫描前端目录，生成带数字 ID 的 `.rc` 资源脚本，与 `WebView2Loader.dll` 一起编译进 exe。

2. **运行时 - DLL 加载**：`EmbeddedDll::Load()` 从 exe 资源段提取 `WebView2Loader.dll` 写入临时文件，调用 `LoadLibraryW()` 加载后立即删除临时文件。`__pfnDliNotifyHook2` delay-load 钩子拦截 MSVC 的延迟加载机制，直接返回已加载的模块句柄。

3. **运行时 - 资源提供**：`SetVirtualHostNameToFolderMapping` 将 `tauricpp.app` 映射为虚拟主机名。`WebResourceRequested` 拦截所有 `https://tauricpp.app/*` 请求，从 `VirtualFS`（内存中）提供内容，自动设置 MIME 类型、CORS 和缓存头。

4. **运行时 - 通信**：`AddScriptToExecuteOnDocumentCreated` 在页面脚本执行前注入桥接 JS。`postMessage` / `WebMessageReceived` 处理 JS→C++ 通信，`ExecuteScript` 处理 C++→JS 通信。

## 对比

| | **TauriCPP** | **Electron** | **Tauri** | **WPF** | **Core UI** |
|---|---|---|---|---|---|
| **后端语言** | C++ | JS/TS | Rust | C#/.NET | C/JS(QuickJS) |
| **渲染引擎** | WebView2 (Edge) | Chromium | WebView2/WebKitGTK | DirectX (.NET) | Direct2D/D3D11 |
| **前端技术** | 任意 Web 框架 | 任意 Web 框架 | 任意 Web 框架 | XAML | .uix（类 Vue SFC） |
| **二进制体积** | ~1-10 MB | 100+ MB | 3-10 MB | 需 .NET 运行时 | ~3 MB DLL |
| **内存占用** | 50-80 MB | 150+ MB | 30-80 MB | 80+ MB | < 30 MB |
| **启动速度** | < 500ms | 1-3s | < 500ms | 0.5-1s | < 200ms |
| **源码保护** | 是（嵌入 exe，内存提供） | 否（asar 可解包） | 有限 | 否（IL 可反编译） | 是（编译为二进制） |
| **单文件部署** | 是 | 否 | 是（打包后） | 否 | 是（单个 DLL） |
| **跨平台** | 仅 Windows | Win/macOS/Linux | Win/macOS/Linux | 仅 Windows | 仅 Windows |
| **运行时依赖** | WebView2（Win10/11 预装） | 捆绑 Chromium | WebView2/WebKitGTK | .NET 运行时 | 无 |
| **后端生态** | C++ 原生库 | npm 生态 | Rust crates 生态 | NuGet 生态 | C ABI，任意语言 |

### TauriCPP vs Electron

Electron 捆绑整个 Chromium 引擎，二进制体积 100+ MB，内存占用 150+ MB。TauriCPP 使用系统自带的 WebView2（Windows 10/11 已预装），二进制体积控制在 10 MB 以内。前端源码嵌入 exe 并从内存提供——不像 asar 归档可以被解包提取。

### TauriCPP vs Tauri

Tauri 是最相似的框架，但需要 Rust 语言。TauriCPP 使用 C++，可直接调用 Windows API 和现有 C++ 代码库，无需 FFI 开销。两者在 Windows 上都使用 WebView2，二进制体积相近。Tauri 支持跨平台（macOS/Linux 通过 WebKitGTK），TauriCPP 仅支持 Windows，但构建工具链更简单。

### TauriCPP vs WPF

WPF 需要 .NET 运行时，使用 XAML 定义 UI。TauriCPP 使用标准 Web 技术（HTML/CSS/JS），可利用庞大的 Web 生态，前端团队技能可直接复用。WPF 应用的 IL 可被反编译，TauriCPP 将前端资源嵌入原生二进制，更难提取。

### TauriCPP vs Core UI

Core UI 是一个原生渲染框架，使用 Direct2D 渲染和自定义 `.uix` 组件格式 + QuickJS 引擎——极其轻量（3 MB，< 30 MB 内存）。但它使用自定义 UI 框架，CSS 支持有限，控件集较小。TauriCPP 使用完整的 Web 平台（任意 CSS、任意 JS 框架、完整 DOM），可访问整个 npm 生态。代价是内存占用较高（WebView2 开销），但 UI 灵活性和开发者熟悉度大幅提升。

## API 参考

### App

```cpp
tauricpp::App::Config config;
config.window_config.title = "我的应用";
config.window_config.width = 1024;
config.window_config.height = 768;
config.window_config.center = true;
config.window_config.resizable = true;
config.window_config.start_url = "https://tauricpp.app/index.html";

tauricpp::App app(config);
app.OnSetup([](tauricpp::App& app) { /* 消息循环前调用 */ });
int exitCode = app.Run();
```

### Bridge

```cpp
auto& bridge = app.GetBridge();

// 注册命令（前端通过 __tauricpp__.invoke('cmd', args) 调用）
bridge.RegisterCommand("cmd_name", [](const nlohmann::json& args) -> nlohmann::json {
    return {{"result", "ok"}};
});

// 向前端推送事件（前端通过 __tauricpp__.listen('event', cb) 监听）
bridge.Emit("event_name", nlohmann::json{{"key", "value"}});
```

### Window

```cpp
auto& window = app.GetWindow();
window.ExecuteJs("console.log('Hello from C++')");
HWND hwnd = window.GetHwnd();
```

### VirtualFS

```cpp
auto& vfs = app.GetFS();

// 注册文件到虚拟文件系统
vfs.RegisterFile("/path/to/file.html", "<h1>Hello</h1>", "text/html");

// 查找文件
tauricpp::VirtualFS::VFile file;
if (vfs.FindFile("/path/to/file.html", file)) {
    // file.data: std::vector<uint8_t>
    // file.mime_type: std::string
}
```

## 构建选项

```powershell
.\build.ps1                    # 构建（Release，Ninja + MSVC）
.\build.ps1 -Clean             # 清理并重新构建
.\build.ps1 -SetupDeps         # 安装依赖（仅首次使用）
```

## 系统要求

- Windows 10/11（WebView2 运行时，大多数系统已预装）
- Visual Studio 2019+（需安装 C++ 桌面开发工作负载）
- CMake 3.20+
- Ninja（随 VS C++ 工作负载安装）
- vcpkg（classic 模式）
- Python 3.7+

## 许可证

MIT
