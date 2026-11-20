# Chromium / Chrome 架构与关键机制

本文以 Chromium 为主线说明整体架构。Chrome 在 Chromium 之上增加了闭源组件（账号同步、Widevine、专有编解码、更新服务等），但进程模型、渲染引擎、网络栈、策略系统与 Chromium 同源。

---

## 1. 三个观察视角

理解 Chromium 不能只看目录，也不能只看进程列表。至少有三个互相独立又关联的视角：

| 视角 | 回答的问题 | 典型入口 |
| --- | --- | --- |
| 源码模块架构 | 功能由哪些代码实现、如何分层 | `src/` 目录、GN 依赖 |
| 运行时进程架构 | 功能跑在哪个进程、边界在哪 | 任务管理器、`ps`、`chrome://process-internals` |
| 配置控制架构 | 谁决定最终行为 | PrefService、Policy、命令行、扩展 |

**关键认知：源码模块 ≠ 运行时进程。** `net/` 是代码模块，Network Service 是进程；一个进程会用到多个模块，一个模块也会被多个进程复用。

---

## 2. 运行时进程架构

### 2.1 主要进程

| 进程 | 职责 | 安全级别 |
| --- | --- | --- |
| Browser Process | 窗口与标签管理、导航决策、权限、配置、进程调度 | 高（可访问系统资源） |
| Renderer Process | 解析 HTML/CSS、执行 JS、布局与绘制 | 低（强沙箱） |
| GPU Process | 图层合成、光栅化、视频解码、WebGL | 中 |
| Network Service | HTTP/HTTPS/QUIC、DNS、代理、TLS、缓存、Cookie | 中（utility 进程） |
| Utility Process | 音频、视频采集、数据解码等独立服务 | 按服务划分 |
| Zygote（Linux） | 预加载运行时，fork 出受沙箱约束的子进程 | 特殊 |
| Crashpad Handler | 崩溃收集 | 独立 |

一个简化视图：

```
                 +----------------------+
                 |   Browser Process    |
                 |  UI / Tabs / Prefs   |
                 +----+------+-----+----+
                      |      |     |
              Mojo    |      |     |   Mojo
        +-------------+      |     +--------------+
        |                    |                    |
+-------v------+   +---------v--------+   +-------v-------+
|  Renderer(s) |   | Network Service  |   |  GPU Process  |
| Blink + V8   |   | proxy/DNS/TLS    |   | cc / viz      |
+--------------+   +------------------+   +---------------+
```

### 2.2 为什么是多进程

1. **安全**：网页是不可信代码。Renderer 运行在沙箱中，不能直接读文件、开 socket；所有系统能力必须经 Browser/Network Service 代理并校验。
2. **稳定**：单个标签崩溃不拖垮整个浏览器。
3. **性能**：渲染、网络、GPU 可并行；多标签可并行执行。
4. **代价**：内存占用高，进程间通信有成本——这是 Chromium 长期被诟病内存大的根源。

### 2.3 Site Isolation

Renderer 不再简单按标签划分，而是按 **SiteInstance**（站点实例）划分：

- 跨站 iframe 尽量放到独立进程
- 导航跨站时可能发生 **process swap**
- 目的：防止不同站点共享地址空间，缓解 Spectre 类侧信道攻击

### 2.4 一次导航的简化流程

以访问 `https://example.com` 为例：

1. **Browser**：地址栏输入 → NavigationController 发起导航 → 检查权限与策略
2. **Network Service**：解析代理配置 → DNS → 建立 TCP/TLS → 发送 HTTP 请求 → 处理重定向与缓存
3. **Browser**：决定是否 process swap，选定 Renderer
4. **Renderer**：Blink 解析 HTML/CSS 构建 DOM/CSSOM → 布局 → 绘制指令
5. **V8**：执行 JavaScript，可能触发 DOM 变更与重新布局
6. **Compositor / GPU**：合成图层，光栅化并上屏

代理配置只影响第 2 步的网络链路，不影响解析与执行——这也是为什么代理故障表现为“所有站点都打不开”，而不是页面渲染异常。

### 2.5 IPC：Mojo

- 所有跨进程调用走 **Mojo**：message pipe（消息）、data pipe（大数据流）、interface broker（接口注册与发现）
- 接口用 `*.mojom` 定义，编译期生成绑定代码
- Browser 是 broker：Renderer 之间不直接通信
- 不可信进程发来的消息必须经过 mojo 校验（validation），防止反序列化漏洞

---

## 3. 源码模块架构

典型目录（简化）：

```
src/
├── base/                 # 基础库：线程、任务、内存、文件、时间
├── mojo/                 # 进程间通信与接口定义
├── net/                  # 底层网络库：socket、HTTP、TLS、DNS、代理
├── services/
│   ├── network/          # Network Service 的实现（URLLoader 等）
│   ├── audio/            # 音频服务
│   ├── video_capture/    # 视频采集服务
│   └── data_decoder/     # 图片/JSON 等解码隔离
├── content/              # 多进程浏览器内核框架
│   ├── browser/          # 导航、站点实例、进程管理
│   ├── renderer/         # 渲染进程侧胶水
│   ├── public/           # 对外 API
│   └── gpu/
├── chrome/               # Chrome 应用层（浏览器产品功能）
│   ├── browser/          # 标签、书签、设置、扩展宿主
│   ├── renderer/
│   └── common/
├── components/           # 可复用功能模块（policy、sync、signin…）
│   └── policy/           # 企业策略：加载、解析、映射到 Pref
├── third_party/blink/    # 渲染引擎：DOM、CSS、布局、绑定
├── v8/                   # JavaScript 引擎
├── gpu/                  # GPU 服务、命令缓冲
├── cc/ + components/viz/ # 合成器与显示合成
├── ui/                   # Views、Aura、gfx
├── sandbox/              # 各平台沙箱实现
├── extensions/           # 扩展系统
├── build/                # GN 构建模板
└── tools/                # 开发与测试工具
```

### 3.1 分层依赖规则

- `content/` 不依赖 `chrome/`：内核可被其他产品复用（Android WebView、Headless 等）
- `components/` 可被 `chrome/` 与 `content/` 共用，但不能反向依赖产品层
- `base/` 是最底层，几乎不依赖其他模块
- 依赖方向出错，GN 会在编译期报错——架构规则被写进了构建系统

### 3.2 构建系统

- **GN** 生成构建图，**Ninja** 执行编译
- `DEPS` 管理第三方依赖与版本
- `mojom`、`grit`（资源）、`blink IDL` 等在编译期生成代码
- 支持 `gn args` 定制（debug/release、组件化、沙箱开关）

---

## 4. 配置与策略体系（本次故障所在层）

### 4.1 配置的多个来源

Chromium 的行为可能同时受以下来源影响：

1. **默认值**：代码内注册
2. **用户偏好**：`Profile/Preferences`（JSON）、`Local State`
3. **扩展设置**：`chrome.proxy`、`chrome.privacy` 等 API
4. **系统设置**：Linux 上是 GSettings（`org.gnome.system.proxy`），非 GNOME 环境才可能回退到环境变量
5. **命令行参数**：如 `--proxy-server`
6. **企业策略（Policy）**：平台策略文件 / 云管理

### 4.2 策略加载链路

```
/etc/opt/chrome/policies/managed/*.json      （Chrome；Chromium 在 /etc/chromium/policies）
                │
                ▼
        ConfigDirPolicyLoader
        ├─ 枚举目录中的普通文件（不递归、不看扩展名）
        ├─ IsTrustedPath：属主与权限校验
        └─ 解析 JSON，按文件名字典序合并
                │
                ▼
          PolicyBundle → PolicyService
                │
                ▼
        PrefService（managed pref，用户不可改）
                │
                ▼
        各功能模块读取有效配置
                │
                ▼
        Network Service 使用代理配置
```

源码位置：`components/policy/core/common/config_dir_policy_loader.cc`

已验证的实现细节：

- `FileEnumerator(path, false, FILES)`：**不递归**子目录
- **不按扩展名过滤**：`local-proxy-7893.json.disabled` 仍会被当作策略解析
- `IsTrustedPath()`：文件必须属于预期用户（root）且不能被 group/other 写入，否则忽略并告警
- 文件按文件名字典序合并，后者的键覆盖前者
- 加载时机：启动时读取，运行期间按策略刷新周期重新加载（默认约 3 小时）

### 4.3 优先级

managed 策略属于最高优先级，会覆盖用户设置、扩展设置和命令行参数。但要注意：Chromium 并没有一条适用于所有功能的“全局固定优先级链”，不同子系统（代理、权限、下载）各有自己的合成逻辑。安全的表述是：

```
managed policy  >  extension / user pref / system / default
```

### 4.4 代理配置的合成

代理相对特殊，有自己的选择逻辑：

```
ProxySettings 策略（若有，强制生效）
        │
        ▼
ProxyConfigService（Linux：GSettings；必要时回退环境变量/命令行）
        │
        ▼
ProxyResolutionService（DIRECT / fixed_servers / PAC）
        │
        ▼
Network Service 建立连接
```

排查入口：

- `chrome://policy`：当前生效策略
- `chrome://net-internals/#proxy`：Effective proxy settings
- `chrome://version`：实际启动命令行
- `chrome://settings/system`：系统代理入口

---

## 5. 关键子系统速览

### 5.1 Blink（渲染引擎）

- DOM、CSSOM、Style、Layout（LayoutNG）、Paint
- 与 V8 通过绑定层交互（IDL 生成）
- 多线程：主线程负责脚本与布局，合成线程负责滚动与动画

### 5.2 V8（JS 引擎）

- Isolate / Context / Handle scope
- 解释器 Ignition + 优化编译器 TurboFan
- 与 Blink 的 GC（Oilpan）协作，跨语言对象生命周期需要特殊处理

### 5.3 网络栈

- `net/`：URLRequest、HttpNetworkTransaction、HttpStream、QUIC、DNS、缓存
- `services/network/`：把网络能力放进独立进程，通过 Mojo 暴露 URLLoaderFactory
- Cookie、缓存、证书校验都在网络侧完成

### 5.4 存储

- Profile 目录：`Preferences`、`History`（SQLite）、`Cookies`（SQLite）、`Local Storage`（LevelDB）
- `Local State`：跨 Profile 的浏览器级配置
- 扩展数据：`Local Extension Settings/<id>`（LevelDB）

### 5.5 图形与合成

- `cc`：图层树、栅格化、动画
- `viz`：显示合成器，跨进程协调
- Skia 绘制，ANGLE 转译 OpenGL ES 到各平台图形 API

### 5.6 扩展系统

- Manifest V2 / V3，V3 以 Service Worker 替代后台页
- `chrome.proxy` 等 API 可修改浏览器级配置，但受策略约束
- 权限模型较粗，扩展仍是主要攻击面之一

### 5.7 沙箱

- Linux：user namespace、PID/mount namespace、seccomp-bpf
- Renderer 无文件系统与网络直接访问权
- Zygote 预加载后 fork，减少沙箱进程启动开销

---

## 6. 从架构看本次故障

把故障映射到分层模型：

| 层 | 本次表现 | 排查手段 |
| --- | --- | --- |
| 启动入口 | desktop 文件带 `--proxy-server` | 读 `.desktop`、看 `/proc/PID/cmdline` |
| 策略层 | `policies/managed` 强制 7893 | `chrome://policy`、读目录 |
| 偏好层 | Preferences 无 proxy 字段 | 解析 Profile JSON |
| 系统层 | GSettings mode=none | `gsettings` |
| 网络层 | Chrome 仍连接 7893 | 监听端口 + `sudo ss -tnp` |
| 传输层 | 端口无监听 → 连接拒绝 | `curl --proxy` 复现 |

两个结论：

1. **错误码已经给出了层次**：`ERR_PROXY_CONNECTION_FAILED` 说明“仍有代理配置且连不上”，不需要从 DNS、路由、防火墙查起。
2. **配置层优先级高于运行时状态**：端口关没关、进程在不在，都不能解释“为什么选这个代理”；必须找到配置来源。

最有效的定位手法不是继续枚举配置文件，而是制造可观测性：

```bash
# 在可疑端口起监听并保持连接
python3 -c "import socket,time;s=socket.socket();s.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1);s.bind(('127.0.0.1',7893));s.listen(50);c,_=s.accept();time.sleep(30)"

# 另一个终端：抓客户端进程
sudo ss -tnp state established '( dport = :7893 )'
```

---

## 7. 值得记住的工程思想

1. **进程边界即安全边界**：不可信代码放进沙箱进程，能力通过 IPC 授予。
2. **模块划分服务于依赖管理**：`content` 与 `chrome` 的边界让内核可复用，规则由构建系统强制执行。
3. **配置必须能回答“谁最终生效”**：多来源配置需要明确的合成与优先级，否则会出现本次这种“改了没用”的假象。
4. **可观测性优先于猜测**：先让系统暴露行为（端口监听、net-internals、tracing），再解释行为。
5. **别用表象修改代替根因修复**：改扩展名禁用策略、重启浏览器重试，都是绕过症状。

---

## 8. 延伸阅读

- [Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)
- [Getting Around the Chromium Source Code](https://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code/)
- [Chromium Source Tree Overview](https://chromium.googlesource.com/chromium/src/+/main/docs/source_tree_overview.md)
- [ConfigDirPolicyLoader 源码](https://chromium.googlesource.com/chromium/src/+/main/components/policy/core/common/config_dir_policy_loader.cc)
