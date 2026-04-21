# HVNC - Hidden Virtual Network Computing (TinyNuke Fork)

> **免责声明**：本项目仅供安全研究、逆向工程学习和教育目的使用。严禁将本项目用于任何未经授权的计算机访问行为。使用者须自行承担全部法律责任。

本项目基于 [TinyNuke](https://github.com/rossja/TinyNuke) 僵尸网络的 HVNC 模块，使用 C++ (MSVC) 实现，版本 1.1.0。包含 **Server（控制端）** 和 **Client（被控端）** 两个独立的 Win32 程序。

---

## 技术原理

HVNC（Hidden Virtual Network Computing）的核心思路是利用 Windows 多桌面机制，通过 `CreateDesktopA` API 创建一个对当前登录用户不可见的独立桌面。在该隐藏桌面上运行的所有程序不会出现在用户的主屏幕、任务栏中。

---

## 整体架构

```
Client（被控端）                              Server（控制端）
     │                                            │
     │── TCP 连接 1 (input) ─── "MELTED" 握手 ──▶│ 分配 Client 槽位，创建控制窗口
     │                                            │ 返回确认后，Client 启动 desktop 线程
     │── TCP 连接 2 (desktop) ─ "MELTED" 握手 ──▶│ 开始帧传输循环
     │                                            │
     │  [屏幕捕获管线]                             │
     │  枚举窗口 → PrintWindow → JPEG压缩          │
     │  → 帧差分 → LZNT1压缩 → 发送帧数据 ───────▶│ RtlDecompressBuffer 解压
     │                                            │ → 差分像素合并 → SetDIBits 渲染
     │                                            │
     │◀─── 鼠标/键盘/命令消息 (msg+wParam+lParam) ─│ 坐标按分辨率比例换算后转发
```

### 通信协议

1. **握手**：Client 向 Server 发送 7 字节魔术值 `"MELTED\0"`，随后发送 `int` 类型的连接类型标识（`0` = desktop，`1` = input）
2. **Client 识别**：Server 通过 `getpeername` 获取 Client IP 地址的 `S_un.S_addr`（32 位整数）作为唯一标识（`uhid`）
3. **input 连接先建立**，Server 分配客户端槽位并创建控制窗口后回复确认；Client 收到确认后在新线程中建立 **desktop 连接**
4. **帧传输循环**：
   - Server 发送 `(width, height)` → Client 捕获并压缩 → Client 发送 `same` 标志
   - 若 `same=1`（无变化），Server sleep 33ms 后请求下一帧
   - 若 `same=0`，Client 依次发送：`screenWidth, screenHeight, pixelsWidth, pixelsHeight, compressedSize, compressedData`
   - Server 解压渲染后回复确认

---

## 屏幕捕获管线（Client 端）

源码位于 `Client/HiddenDesktop.cpp`，`GetDeskPixels()` 和相关函数实现了完整的捕获流程：

| 步骤 | 实现 | 代码位置 |
|------|------|----------|
| 1. 枚举窗口 | `EnumWindowsTopToDown()` 从底层到顶层遍历隐藏桌面上所有可见窗口 | `HiddenDesktop.cpp:90-98` |
| 2. 逐窗口绘制 | 对每个窗口调用 `PrintWindow()` 绘制到内存 DC，再 `BitBlt` 合成到全屏 DC | `HiddenDesktop.cpp:61-88` |
| 3. 分辨率缩放 | 若 Server 请求尺寸小于实际桌面，使用 `StretchBlt` + `HALFTONE` 模式缩放 | `HiddenDesktop.cpp:148-163` |
| 4. JPEG 有损压缩 | `BitmapToJpg()` 通过 GDI+ 将位图编码为 JPEG 再解码回 24bit RGB，降低像素数据量 | `HiddenDesktop.cpp:36-59` |
| 5. 帧差分 | 将当前帧与上一帧逐像素对比，未变化的像素标记为魔术色 `RGB(255,174,201)` | `HiddenDesktop.cpp:190-226` |
| 6. 魔术色冲突处理 | 若原始像素恰好等于魔术色，将 G 通道 +1 避免误判 | `HiddenDesktop.cpp:194-199` |
| 7. LZNT1 无损压缩 | `RtlCompressBuffer(COMPRESSION_FORMAT_LZNT1)` 对差异帧再次压缩 | `HiddenDesktop.cpp:288-300` |

位图格式：24 位 RGB（`biBitCount=24`, `biCompression=BI_RGB`），无 Alpha 通道。

---

## 输入处理（Client 端）

`InputThread()` (`HiddenDesktop.cpp:518-837`) 接收 Server 转发的 Windows 消息，实现了完整的远程交互：

### 鼠标操作
- **基础消息**：`WM_LBUTTONDOWN/UP`, `WM_RBUTTONDOWN/UP`, `WM_MBUTTONDOWN/UP`, `WM_MOUSEMOVE`, `WM_MOUSEWHEEL`, 双击事件
- **窗口关闭**：鼠标释放时发送 `WM_NCHITTEST`，返回 `HTCLOSE` 则 `PostMessage(WM_CLOSE)`
- **最小化/最大化**：`HTMINBUTTON` → `SC_MINIMIZE`，`HTMAXBUTTON` → 判断当前状态切换 `SC_MAXIMIZE`/`SC_RESTORE`
- **窗口拖拽**：`WM_MOUSEMOVE` + 左键按下时，根据 `WM_NCHITTEST` 返回值（`HTCAPTION`, `HTTOP`, `HTBOTTOM`, `HTLEFT`, `HTRIGHT` 及四个角）计算偏移，调用 `MoveWindow` 实现移动和缩放
- **开始菜单**：检测点击 `"Button"` 窗口类，发送 `BM_CLICK`
- **右键菜单**：检测 `"#32768"` 窗口类（系统菜单），通过 `MN_GETHMENU` + `MenuItemFromPoint` + `VK_RETURN` 完成菜单项选择
- **子窗口定位**：递归调用 `ScreenToClient` + `ChildWindowFromPoint` 定位最深层子窗口

### 键盘操作
- `WM_CHAR`：可打印字符输入
- `WM_KEYDOWN/UP`：方向键、Home/End、PageUp/PageDown、Insert、Enter、Delete、Backspace
- 控制字符（`iscntrl`）在 Server 端被过滤，不转发

### 远程应用启动
通过自定义消息（`WM_USER+1` ~ `WM_USER+8`）触发，所有进程的 `STARTUPINFO.lpDesktop` 指向隐藏桌面名称：

| 消息 | 功能 | 实现细节 |
|------|------|----------|
| `startExplorer` | 启动资源管理器 | 设置注册表 `TaskbarGlomLevel=2`（不合并任务栏），启动 `explorer.exe`，等待 `Shell_TrayWnd` 出现后设置 `ABS_ALWAYSONTOP`，再恢复注册表原值 |
| `startRun` | 打开运行对话框 | `rundll32.exe shell32.dll,#61` |
| `startChrome` | 启动 Chrome | 复制 `%LOCALAPPDATA%\Google\Chrome\User Data\` 到以 BotID 命名的新目录，启动参数 `--no-sandbox --disable-gpu --user-data-dir=<cloned>` |
| `startFirefox` | 启动 Firefox | 解析 `%APPDATA%\Mozilla\Firefox\profiles.ini` 获取 Profile 路径，复制整个 Profile 目录，启动参数 `-no-remote -profile <cloned>` |
| `startEdge` | 启动 Edge | `msedge.exe --no-sandbox --disable-gpu --user-data-dir=...` |
| `startBrave` | 启动 Brave | 先 `killproc("brave.exe")` 终止已有进程，再以相同参数启动 |
| `startIexplore` | 启动 IE | 直接启动 `iexplore.exe` |
| `startPowershell` | 启动 PowerShell | 启动参数 `-noexit -command "[console]::windowwidth=100;..."` 固定窗口大小 |

所有浏览器通过 `cmd.exe /c start` 间接启动。Chrome 和 Firefox 会克隆用户 Profile 以保留已登录会话、Cookie、历史记录。

---

## Server 端实现

### 连接管理 (`Server/Server.cpp`)

- TCP 监听，`accept` 后为每个连接创建独立线程（`ClientThread`）
- 最多同时支持 **256** 个 Client（`gc_maxClients`）
- 每个 Client 占用一个 `Client` 结构体槽位，包含两个 socket（input/desktop）、控制窗口句柄、像素缓冲区等
- 使用全局 `CRITICAL_SECTION` 保护共享的 `g_clients` 数组

### 画面渲染

- 接收压缩帧后，`RtlDecompressBuffer` 解压
- 差分合并：遍历像素，魔术色 `RGB(255,174,201)` 的像素保留上一帧数据，其他像素更新
- `SetDIBits` 写入 HBITMAP → `BitBlt` 绘制到控制窗口的 HDC
- `InvalidateRgn` 触发 `WM_PAINT` 重绘
- 窗口最小化时 `ResetEvent` + `WaitForSingleObject(5000)` 暂停帧请求

### 控制窗口 (`Server/ControlWindow.cpp`)

- 窗口类名 `"HiddenDesktop_ControlWindow"`，标题格式 `"Desktop@<IP> | HVNC - Tinynuke Clone [Melted@HF]"`
- 支持双击事件（`CS_DBLCLKS`）
- 最小窗口尺寸 800x600，最大不超过 Client 实际屏幕分辨率
- 鼠标坐标按 `screenWidth/pixelsWidth` 比例换算后转发
- 系统菜单（右键标题栏）添加应用启动菜单项和全屏选项（全屏功能已注释）

### 输入转发

- 鼠标消息：坐标按比例缩放后，通过 input socket 发送 `(msg, wParam, lParam)` 三元组
- 键盘消息：仅转发特定虚拟键码（方向键、功能键等），`WM_CHAR` 过滤控制字符
- `WM_MOUSEMOVE` 仅在左键按下时转发（避免空闲时大量消息）

---

## Client 端辅助模块

### 动态 API 加载 (`common/Api.h` + `Api.cpp`)

Client 端所有 Windows API 调用均通过动态加载的函数指针完成，避免静态导入表暴露调用关系：

- `InitApi()` 在启动时通过 `LoadLibraryA` + `GetProcAddress` 从 **14 个 DLL** 中解析约 **160 个** API 函数指针
- 涉及 DLL：kernel32, ntdll, user32, gdi32, ws2_32, msvcrt, shell32, shlwapi, advapi32, secur32, version, psapi, wininet
- 字符串使用 `ENC_STR_A ... END_ENC_STR` 宏包裹（当前实现为直接透传，预留了 XOR 混淆接口）

### BotID 生成 (`common/Utils.cpp:17-42`)

- 获取 Windows 安装分区的卷序列号（`GetVolumeInformationA`）作为伪随机种子
- 通过线性同余算法 `seed = 1352459 * seed + 2529004207` 生成 GUID
- 格式化为 `"%08lX%04lX%lu"` 字符串，长度 ≤35 字节
- 同一台机器生成的 BotID 始终相同（确定性），用于：隐藏桌面命名、浏览器 Profile 克隆目录命名

### C2 面板通信 (`common/Panel.cpp` + `HTTP.cpp`)

Client 包含完整的 C2 通信基础设施（当前 HVNC 模式未使用，但代码完整）：

- **HTTP 客户端**：手写 socket 实现，支持 `GET`/`POST`、`Content-Length` 和 `Transfer-Encoding: chunked` 两种响应模式
- **面板协议**：`POST` 请求到 `/panel/client.php?<BotID>`，首次请求 `"ping"` 获取 XOR 密钥，后续请求/响应均使用该密钥进行 XOR 混淆
- **多主机轮换**：`host[]` 数组支持多个 C2 地址，连接失败时自动切换（当前仅配置 `127.0.0.1`）
- **默认配置**：`Common.h` 中 `HOST=127.0.0.1`, `PORT=80`, `POLL=60000`（轮询间隔 60 秒）

### 持久化与进程注入 (`common/Utils.cpp`)

代码中包含以下功能（主要供 TinyNuke 完整版使用）：

- **注册表持久化**：`SetStartupValue()` 通过 `NtOpenKey` + `NtSetValueKey` 写入 `HKCU\...\Run` 键实现开机自启
- **安装路径**：`GetInstallPath()` 生成 `%APPDATA%\<BotID>\<BotID>.exe`
- **进程空洞注入（Process Hollowing）**：`BypassTrusteer()` 以挂起方式创建目标进程，替换内存映像后恢复执行
- **DLL 下载与缓存**：`GetDlls()` 从 C2 面板下载 x86/x64 DLL，XOR 加密存储到 `%TEMP%\<BotID>32`/`64`
- **Firefox 设置修改**：`SetFirefoxPrefs()` 向 `prefs.js` 注入配置，禁用 SPDY/HTTP2、多进程、硬件加速
- **IE 设置修改**：`DisableMultiProcessesAndProtectedModeIe()` 通过注册表禁用 IE 多进程和保护模式
- **目录递归复制**：`CopyDir()` 手写实现，用于浏览器 Profile 克隆

---

## 项目结构

```
HVNC-main/
├── HVNC.sln                    # VS2017+ 解决方案（2 个 C++ 项目）
├── README.md
├── Client/                     # 被控端
│   ├── HVNC.vcxproj            # 项目文件（依赖 Server 项目构建顺序）
│   ├── Main.cpp                # 入口：隐藏控制台窗口，配置 host/port，启动隐藏桌面
│   ├── HiddenDesktop.cpp       # 核心：隐藏桌面创建、屏幕捕获管线、输入处理、应用启动
│   └── HiddenDesktop.h
├── Server/                     # 控制端
│   ├── Server.vcxproj          # 项目文件（仅 Win32 平台）
│   ├── _version.h              # 版本号 1.1.0
│   ├── Main.cpp                # 入口：WinMain + AllocConsole，提示输入监听端口
│   ├── Server.cpp              # 核心：TCP accept、Client 管理、帧解压合并渲染、输入转发
│   ├── Server.h
│   ├── ControlWindow.cpp       # Win32 控制窗口的注册与创建
│   ├── ControlWindow.h
│   └── Common.h                # Server 专用头文件
├── common/                     # Client 与 Server 共享代码
│   ├── Common.h                # 公共头文件，定义 HOST/PORT/POLL 等常量
│   ├── Api.h                   # 150+ Windows API 函数指针类型定义与声明
│   ├── Api.cpp                 # InitApi()：动态加载 14 个 DLL 的所有函数指针 + 字符串初始化
│   ├── Utils.h                 # 工具函数声明
│   ├── Utils.cpp               # BotID 生成、XOR 混淆、目录复制、注册表持久化、进程空洞注入等
│   ├── HTTP.h                  # HTTP 请求数据结构定义
│   ├── HTTP.cpp                # 手写 HTTP/1.1 客户端（socket 实现，支持 chunked 编码）
│   ├── Panel.h                 # C2 面板通信接口
│   ├── Panel.cpp               # C2 通信：ping 握手获取密钥、XOR 加密请求/响应、多主机轮换
│   └── Inject.h                # DLL 注入接口声明（InjectDll，实现未包含在当前代码中）
├── _bin/Release/Win32/         # 编译输出
│   ├── Client.exe
│   └── Server.exe
└── _tmp/                       # 编译中间文件
```

---

## 编译指南

### 环境要求

| 项目 | 要求 | 说明 |
|------|------|------|
| 操作系统 | Windows 10 / Server 2016 及以上 | 编译和运行均需 Windows |
| IDE | Visual Studio 2022（推荐） | 2017/2019 亦可，需手动调整 PlatformToolset |
| 工作负载 | **使用 C++ 的桌面开发** | VS Installer 中勾选 |
| Windows SDK | 10.0（任意版本） | 随 VS 工作负载自动安装 |
| 平台工具集 | v143（VS2022）或 v142（VS2019） | 项目已配置 v143 |
| 目标平台 | **Win32 (x86)** | Server 仅支持 Win32；Client 支持 Win32 和 x64 |
| 外部依赖 | 无 | GDI+、Winsock、ntdll 均为系统自带 |

### 两个项目的编译差异

| | Client (HVNC.vcxproj) | Server (Server.vcxproj) |
|---|---|---|
| 子系统 | Console (`/SUBSYSTEM:CONSOLE`) | Windows (`/SUBSYSTEM:WINDOWS`) |
| 入口函数 | `main()` | `WinMain()` |
| 字符集 | MultiByte | Unicode |
| 运行时库 | `/MT`（静态链接 CRT） | `/MT`（静态链接 CRT） |
| C++ 异常 | 禁用 (`/EHsc` 关闭) | 默认 |
| Release 优化 | `/O1`（最小体积） | `/O2`（最大速度） |
| 额外链接库 | GDI+（`Gdiplus.lib`，通过 `#pragma comment` 自动链接） | 无额外 |
| 构建依赖 | 依赖 Server 项目先构建 | 无依赖 |
| 可用平台 | Win32, x64 | 仅 Win32 |

### 编译前配置 — 修改 Server IP / 端口

> **重要**：Client 的连接目标地址是**编译时硬编码**的，必须在编译前修改源码。Server 端口则在运行时输入，无需改代码。

要让 Client 连接到你自己的 Server，需要修改 **2 个文件共 3 处**：

#### 必改项：Client 连接地址

**文件 `Client/Main.cpp` 第 17-18 行**：

```cpp
const char* host = "127.0.0.1";  // ← 改为你的 Server IP 或域名
const int port = strtol("4043", nullptr, 10);  // ← 改为你的 Server 监听端口
```

例如，要连接到公网 IP `203.0.113.50` 的 `8888` 端口：

```cpp
const char* host = "203.0.113.50";
const int port = strtol("8888", nullptr, 10);
```

#### 可选项：C2 面板地址

如需使用 C2 面板通信（当前 HVNC 模式不使用），需同步修改以下两处：

**文件 `common/Common.h` 第 23-26 行**：

```cpp
#define HOST (char*)"127.0.0.1"  // ← C2 面板地址
#define PORT 80                  // ← C2 面板端口
#define POLL 60000               // ← 轮询间隔（毫秒）
```

**文件 `common/Api.cpp` 第 484 行**：

```cpp
Strs::host[0] = ENC_STR_A"127.0.0.1"END_ENC_STR;  // ← 与 Common.h 中的 HOST 保持一致
```

#### 汇总

| 修改内容 | 文件位置 | 行号 | 是否必须 |
|----------|----------|------|----------|
| Server IP | `Client/Main.cpp` | 17 | **是** |
| Server 端口 | `Client/Main.cpp` | 18 | **是** |
| C2 面板 IP | `common/Common.h` | 23 | 否（HVNC 模式不用） |
| C2 面板端口 | `common/Common.h` | 25 | 否 |
| C2 面板 IP（字符串） | `common/Api.cpp` | 484 | 否（需与 Common.h 同步） |

> **Server 端口**无需编译时修改 — 运行 `Server.exe` 后会在控制台提示输入监听端口。

### 方法一：Visual Studio IDE 编译

**步骤 1 — 安装 Visual Studio**

下载 [Visual Studio 2022 Community](https://visualstudio.microsoft.com/downloads/)，在 Visual Studio Installer 中勾选：

- [x] **使用 C++ 的桌面开发**（Desktop development with C++）

确保以下组件已安装（通常随工作负载自动勾选）：

- MSVC v143 - VS 2022 C++ x64/x86 生成工具
- Windows 10/11 SDK（任意版本）

**步骤 2 — 打开解决方案**

双击 `HVNC.sln`，Visual Studio 加载后：

- 若提示"重定目标项目"（Retarget Projects），选择已安装的 Windows SDK 版本和平台工具集，点击确定
- 若提示升级 PlatformToolset，接受升级到 v143

**步骤 3 — 选择构建配置**

在工具栏中设置：

- 解决方案配置：**Release**
- 解决方案平台：**Win32**

> Server 项目仅有 Win32 平台配置。若选择 x64，Server 将跳过构建，仅编译 Client。

**步骤 4 — 生成解决方案**

菜单 → **生成** → **生成解决方案**（快捷键 `Ctrl+Shift+B`）

构建顺序：Server → Client（由 `.sln` 中的 `ProjectDependencies` 定义）。

**验证编译成功**：输出窗口显示 `========== 生成: 成功 2 个 ==========`

### 方法二：MSBuild 命令行编译

适用于无 IDE 环境或 CI/CD 场景。

**步骤 1 — 安装 Build Tools**

下载安装 [Build Tools for Visual Studio 2022](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022)，勾选：

- [x] **C++ 生成工具**（C++ build tools）

**步骤 2 — 打开开发者命令提示符**

从开始菜单启动 **Developer Command Prompt for VS 2022**（或 **x86 Native Tools Command Prompt**）。

> 也可以在普通 CMD 中手动初始化环境：
> ```cmd
> "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x86
> ```

**步骤 3 — 执行编译**

```cmd
cd C:\path\to\HVNC-main

:: 编译 Release|Win32（推荐）
msbuild HVNC.sln /p:Configuration=Release /p:Platform=Win32 /m

:: 或分别编译单个项目
msbuild Server\Server.vcxproj /p:Configuration=Release /p:Platform=Win32
msbuild Client\HVNC.vcxproj /p:Configuration=Release /p:Platform=Win32
```

参数说明：
- `/p:Configuration=Release` — 使用 Release 配置（优化开启，无调试符号）
- `/p:Platform=Win32` — 目标平台 x86
- `/m` — 多核并行编译，加速构建

**编译 Debug 版本**（含调试符号，便于调试）：

```cmd
msbuild HVNC.sln /p:Configuration=Debug /p:Platform=Win32 /m
```

**编译 x64 Client**（Server 不支持 x64）：

```cmd
msbuild Client\HVNC.vcxproj /p:Configuration=Release /p:Platform=x64
```

### 方法三：CMake 交叉编译（高级）

项目未提供 CMakeLists.txt，但如果需要从 CMake 驱动编译，可手动调用 MSBuild：

```cmake
execute_process(
  COMMAND msbuild HVNC.sln /p:Configuration=Release /p:Platform=Win32 /m
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)
```

### 输出文件

编译成功后，生成文件位于：

```
_bin\<Configuration>\<Platform>\
├── Client.exe    (~117 KB, Release|Win32)
├── Client.pdb    (调试符号)
├── Server.exe    (~260 KB, Release|Win32)
└── Server.pdb    (调试符号)
```

中间文件位于 `_tmp\<Configuration>\<ProjectName>\<Platform>\`。

### 编译问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `error MSB8020: 未找到 v143 的生成工具` | 未安装对应版本的 MSVC 工具集 | VS Installer 中安装 "MSVC v143" 组件；或在项目属性 → 常规 → 平台工具集中改为已安装的版本 |
| `error MSB8036: 找不到 Windows SDK` | 未安装 Windows 10 SDK | VS Installer 中安装 "Windows 10 SDK"；或右键解决方案 → 重定目标项目 |
| `fatal error C1083: 无法打开包括文件: "gdiplus.h"` | Windows SDK 未正确安装 | 重新安装 "使用 C++ 的桌面开发" 工作负载 |
| `error LNK2019: 无法解析的外部符号 _GdiplusStartup` | GDI+ 库未链接 | 正常不会出现（已通过 `#pragma comment(lib,"Gdiplus.Lib")` 自动链接）；若出现，在项目属性 → 链接器 → 输入 → 附加依赖项中添加 `Gdiplus.lib` |
| `warning C4996: 'GetVersionExA' 已弃用` | `GetVersionExA` 在 Win 8.1+ 被弃用 | 可忽略，不影响编译和运行 |
| 仅编译出 Client，没有 Server | 选择了 x64 平台 | Server 仅支持 Win32，请切换到 Win32 平台 |
| Debug 编译时链接错误 | Debug|Win32 使用 v142 工具集 | 在项目属性中将 Debug 配置的平台工具集也改为 v143，或安装 v142 工具集 |

### 清理构建

```cmd
:: 清理所有编译产物
msbuild HVNC.sln /t:Clean /p:Configuration=Release /p:Platform=Win32

:: 或手动删除输出目录
rmdir /s /q _bin _tmp
```

---

## 配置说明

### Client 端

编译前修改 `Client/Main.cpp`：

```cpp
const char* host = "127.0.0.1";  // Server IP 地址
const int port = strtol("4043", nullptr, 10);  // Server 端口
```

Client 启动时会立即隐藏控制台窗口（`ShowWindow(GetConsoleWindow(), SW_HIDE)`）。

### Server 端

运行时通过控制台交互输入监听端口，无需编译时固定。

### C2 面板（可选）

如需使用 C2 面板功能，修改 `common/Common.h`：

```cpp
#define HOST (char*)"your-c2-server.com"
#define PORT 80
#define POLL 60000  // 轮询间隔，毫秒
```

同时修改 `common/Api.cpp` 第 484 行的 `Strs::host[0]`。

---

## 控制端功能菜单

Server 控制窗口的系统菜单（右键标题栏）提供以下操作：

| 菜单项 | 功能 | 实现说明 |
|--------|------|----------|
| Start Explorer | 启动资源管理器（含任务栏） | 设置任务栏不合并，启动后等待 Shell_TrayWnd 出现并置顶 |
| Run... | 打开运行对话框 | `rundll32.exe shell32.dll,#61` |
| Start Powershell | 启动 PowerShell | 固定窗口尺寸 100x30 |
| Start Chrome | 克隆 Profile 后启动 | 复制 User Data 目录，禁用 sandbox/GPU |
| Start Edge | 启动 Edge | 禁用 sandbox/GPU |
| Start Brave | 启动 Brave | 先终止已有 brave.exe 进程 |
| Start Firefox | 克隆 Profile 后启动 | 解析 profiles.ini 定位 Profile 目录 |
| Start Internet Explorer | 启动 IE | 直接启动 |

---

## 已知限制

1. **Server 仅 Win32 平台**：`Server.vcxproj` 未配置 x64 构建目标
2. **Client 数量上限**：最多 256 个并发连接，受 `g_clients` 静态数组限制
3. **PrintWindow 局限**：依赖 GDI 渲染，对使用 DirectX/硬件加速的应用（如游戏、部分现代浏览器页面）效果有限
4. **版本检测过时**：`GetVersionExA` 在 Win 8.1+ 已被弃用，`dwMajorVersion < 6` 的子窗口递归枚举分支在现代系统上永远不会执行
5. **单 IP 识别**：使用 Client IP 的 `S_un.S_addr` 作为唯一标识，NAT 后同一出口 IP 的多个 Client 会冲突
6. **无加密传输**：屏幕数据和控制命令通过明文 TCP 传输，仅 C2 面板通信使用 XOR 混淆
7. **Inject.h 声明但未实现**：`InjectDll()` 函数仅有头文件声明，实现代码未包含在当前项目中

---

## 技术参考

- 原始项目：[rossja/TinyNuke](https://github.com/rossja/TinyNuke)
- MITRE ATT&CK：[T1564.006 - Hidden Window](https://attack.mitre.org/techniques/T1564/006/)
- Windows API：[CreateDesktopA](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-createdesktopa)、[PrintWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-printwindow)
- 压缩算法：[RtlCompressBuffer (LZNT1)](https://learn.microsoft.com/en-us/windows/win32/api/compressapi/nf-compressapi-compress)
