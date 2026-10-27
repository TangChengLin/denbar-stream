# Denbar 两端串流集成设计

日期：2026-10-27。服务端采用用户已确认的 C++23 + Sunshine 轻量 Fork；客户端直接集成到 denbar-shell。本文定义一期局域网的实现边界，并保留二期厂商 SDK 接入点。硬件性能需要独立验收，仓库准备完成不代表串流功能已经实现。

## 1. 项目与基线

| 项目 | 位置 | 基线与职责 |
| --- | --- | --- |
| denbar-stream | `E:/VisualStudioProjects/denbar-stream`；GitHub `TangChengLin/denbar-stream` | Sunshine Fork；C++23；Windows x64 采集、编码、发送、输入注入及管理 |
| denbar-shell | `E:/QtProjects/denbar-shell`；原 Cursor 远程 | Qt Quick 6.8.3、C++17、Linux ARM64；接收、MPP 解码、音频、Qt 呈现、外设与现有 UI |

本次源码核对基线：Sunshine `d0b4a88ef727f8dabf80cc970a14739b55c885de`；denbar-shell 本地 master `214abf23acda2960bc05dd63bbaf020bd5a5fa2f`。客户端 origin fetch 在本轮因 SSH 公钥认证失败；本地 origin/master 与 master 相同，但不能据此声称远程最新。

服务器以该 Sunshine 提交作为开发基线，不把 master 标签当发布资格。正式版本需要通过本文验收后记录精确 commit、子模块和依赖清单。已发现的官方正式版 `v2026.914.233613` 可作为对照，不能把它的文件行号、测试清单与当前 master 混用。

两个仓库从各自 master 建立 `codex/denbar-stream-integration`。设计与计划保存在 `docs/superpowers/`，本轮只准备仓库和文档。两端 C++ 标准彼此独立，不共享编译后的 C++ ABI。

## 2. 目标与范围

- 一期公司有线 LAN，Windows 主机适配 Intel UHD 730、Nvidia RTX 30 和 RTX 50 的实际设备；一台主机同一时刻只允许一个活动串流连接。
- 产品目标为 2560×1440、240 个不同视频画面每秒、显示输出 240Hz，包含音频、键盘、相对鼠标和手柄。软件兼容与性能达标分别报告，低档联调不代替 240 验收。
- PC 云电脑【开始】是一期真实生产入口。保留现有游戏中菜单、剩余时间提示、返回本地桌面、继续/恢复和退出交互；现有 TV 与 visual-state 夹具不回归。
- 一期使用 HEVC Main、8-bit、4:2:0、SDR 作为共同高帧率候选；H.264 作为兼容档，不能默认承诺同样吞吐。HDR、AV1、多用户 GPU 分租、USB 透传、麦克风上传不进入一期。
- 二期由厂商 Linux ARM64 SDK 负责 WebRTC 会话和公网传输，回调编码媒体；denbar-shell 复用本地媒体与 UI。denbar-stream 不承担该二期厂商链路的服务器功能。

## 3. 方案取舍

| 方案 | 取舍 | 决策 |
| --- | --- | --- |
| 独立管理程序 + Sunshine 二进制 | 引擎升级简单，直接修改采集编码需要另建下游 | 曾考虑，未选 |
| Sunshine 轻量 Fork | 能维护实际编码选择、诊断、安装及兼容补丁；承担上游合并成本 | 用户已选 |
| 重写 Windows 采集、编码、传输栈 | 可控范围大，重复实现配对、输入、丢包恢复和设备兼容 | 不选 |

客户端采用仓内 Moonlight 协议适配 + 共享 MPP/音频/Qt 呈现，不另建客户端仓库。视频与 Qt 界面由同一个显示系统合成；不启动抢占显示的独立 KMS 播放器。

## 4. 总体链路

```mermaid
flowchart LR
  W[Windows 交互桌面与音频] --> S[denbar-stream / Sunshine]
  S --> E[Quick Sync 或 NVENC]
  E --> M[Moonlight 协议与 LAN]
  M --> B[denbar-shell MoonlightBackend]
  V[二期厂商 WebRTC SDK] --> A[VendorSdkBackend]
  B --> D[MPP 视频解码 / Opus 音频解码]
  A --> D
  D --> Q[Qt Quick 视频节点 + 现有菜单提示]
  Q --> O[显示与音频输出]
  I[键盘 鼠标 手柄] --> R[输入所有权路由]
  R --> B
  R --> A
```

## 5. Windows 服务端

### 5.1 上游复用与构建

保持现有 `src/platform/windows/`、`src/video.cpp`、`src/nvenc/`、`src/audio.cpp`、`src/nvhttp.cpp`、`src/rtsp.cpp`、`src/stream.cpp` 和输入层。配对、证书、协议、加密与 FEC 沿用上游。一期不修改 Moonlight 线协议。

遵循仓库 AGENTS：Windows 使用 MSYS2 UCRT64、CMake、Ninja，构建目录 `cmake-build-*`；当前主 C++ 标准为 23，第三方/CUDA 保留其自身要求。Qt 和原生 Node.js 跟随所选上游依赖，不强制客户端的 Qt 6.8.3，也不预设 MSVC 已受支持。

复用上游服务安装与交互会话捕获方式；一期服务捕获基线 `capture=ddx`。不让自有 Session 0 服务直接采集交互桌面。锁屏、注销、显示器拔插均产生明确状态，恢复时重新验证输出和编码设备。

### 5.2 最小新增模块

新增 `src/denbar/profile.{h,cpp}`、`session_policy.{h,cpp}`、`diagnostics.{h,cpp}`，必要接缝修改保持小范围。管理能力复用 `src/confighttp.cpp` 现有认证/CSRF 保护与现有 Vue 管理界面；不另建管理网络服务。

Profile 独立存储为 `denbar-profiles.json`，schemaVersion=1。字段包含 id、revision、capture、encoder、adapterName、outputId、requireHardware、allowExperimental、allowedModes。allowedModes 每项为 codec、width、height、fps、maxBitrateKbps、qualification（Experimental/Verified）、qualificationId 的完整组合。encoder 值为 `quicksync` 或 `nvenc`，不是 `qsv`。id 唯一；尺寸、fps、码率必须为正且在候选范围内；未知字段/版本明确拒绝。应用 profile 时生成 Sunshine 已支持的配置字段，不能把所有 profile 字段原样写入 sunshine.conf。

参数权威明确分层：profile 决定采集设备、编码器与允许的媒体组合；客户端在允许组合中提出具体 codec/width/height/fps/bitrate 请求。尺寸与帧率必须精确命中一项 allowedModes，码率不超过该项上限；launch 和 RTSP 协商分别校验它们所含的参数，全部通过才激活 lease。冲突、过期能力清单或编码器实际结果不匹配时明确拒绝，不覆盖客户端选择，也不静默降档。重启成功仅表示配置载入，不能当作后续请求或性能已通过。

一期能力清单通过服务端管理界面导出无密钥的 host-profile.json（schemaVersion、hostId、profileRevision、allowedModes），部署到客户端 SettingsStore；地址由安装配置提供，配对证书单独管理。它用于设置项过滤和启动前校验，服务端仍是最终权威；不让客户端持有服务器管理密码，不为此改变 Moonlight 线协议。过期清单导致拒绝时提示更新配置；后续可以另加经过认证的能力同步接口，但不属于一期必要链路。

profileRevision是本地配置关联与诊断字段，不假定Moonlight请求携带它。客户端可拒绝请求与本地清单revision不一致；服务端依据当前allowedModes校验实际协商参数。旧清单中的组合仍被服务端允许时可以连接，只有实际冲突才拒绝；不能声称在不扩展协议的情况下比较了两端revision。

一期开关配置只允许在无 pending/active 会话时应用：申请配置事务锁、验证、暂存、原子保存并受控重启；重启失败可恢复上一个配置。配对、凭据与会话状态文件不能被配置覆盖。与 launch/resume 共用 admission 锁，避免“检查空闲后又接入新会话”的竞争。

初始联调档位为 1080p60、1440p120；目标档为 1440p240，码率调试起点建议 100000 Kbps（100 Mbps），再按画质和网络实测调整。所有数值都是请求值；只有测量记录匹配主机、驱动、输出和客户端能力后，240 档才标为已验证。

候选档位最初均为Experimental，验收环境可以显式启用allowExperimental来测试，界面始终标记未验证。正式部署默认false，只开放Verified且附qualificationId的组合。这样允许对240做首次端到端测试，同时不把可选档位等同于已通过性能验收。

GPU/显示选择复用 `display_device` 与 DXGI 发现。现有 adapter_name 是描述字符串，不冒称是稳定唯一 LUID；若同名设备无法确定则报告歧义，不静默猜测。配置的显示、渲染 GPU、采集 GPU、实际编码器分别记录。UHD 730 可用硬编不代表能以 240fps 渲染目标游戏。

上游指定 encoder 探测失败后可能自动选择其它编码器。一期 Denbar 严格硬编模式必须核对实际结果：指定硬编不可用即明确拒绝，不把软件回退报成成功。NVENC/Quick Sync 调参沿用现有选项，低延迟目标为避免重排序和长前瞻，以实际输出验证。

### 5.3 单活动流与恢复

新增线程安全 admission lease，状态为 idle → reserved → active → idle。launch/resume 在改变显示/应用前取得 lease；RTSP 成功后转 active；失败、握手超时、取消、断连和停止全部释放且幂等。当前上游不存在可直接解决此需求的 channels=1 配置，不能只查询 session_count 后放行。

连接断开后远端桌面/游戏可继续运行，但传输 lease 释放。恢复必须以已配对身份重新获得 lease 并复用应用，另一客户端占用时返回 busy，不自动抢占或取消对方应用。一期的 quit 表示停止本地流，不调用关闭远端应用或 Windows 关机。

### 5.4 部署和同步上游

origin 为用户 Fork，upstream 为官方 Sunshine。保留历史、许可证和第三方声明；Denbar 模块独立，避免全树替换 Sunshine 名称。升级放在 `codex/sync-sunshine-*` 分支，合并选定提交、更新固定子模块、重新验证再发布；不自动部署移动的 master。

安装包应检测已有 Sunshine，避免相同服务、端口和配置同时运行。一期采用单实例部署；发现冲突时提示管理员处理，不覆盖既有配对资料。发布包记录 publisher、版本、二进制哈希、驱动/输入依赖、回退步骤。当前上游 libvirtualhid 最低 broker 版本为 2026.914.1218.10，键鼠/手柄功能按实际驱动组合验收。

## 6. denbar-shell 客户端

### 6.1 代码接缝

`PcCloudPage.startRequested` 经 WindowFrame、WindowManagerLayer 到 PcMain，再调用 GameSessionVM/StreamingController。当前外层缺少 startRequested 转发；生产入口不得调用带假时长的 fixture start。

GameSessionVM 继续是 QML 会话门面；SessionVM 是认证对象，不承担串流状态。新增 `src/streaming/` 内部库；StreamVideoItem 注册到现有 Denbar.Common，GameSessionVM 仍归现有 viewmodel 模块，遵守单次 moc 和 QML 注册约束。

MoonlightHostClient 负责主机地址/发现、配对证书、应用列表、launch/resume；MoonlightBackend 负责 common-c 会话、媒体与输入。初期使用配置的 host 地址和从应用列表获取的 Desktop appId，不硬编码 appId。配对 PIN 通过上游配对界面完成；客户端保存自身身份和配对服务器证书，后续连接验证身份，不能全局忽略 TLS 错误。

### 6.2 会话、预算与后台

内部状态 Idle/Connecting/Playing/Reconnecting/Stopping/Failed；每次开始生成新的 generation。所有回调带 generation，迟到回调不得复活已取消会话；重复开始幂等。stageVisible 与 running 分离，使 Connecting 时可创建视频节点并提交首帧，避免“未 running 不渲染，未渲染又不能 running”的死循环。首个有效呈现回调后才进入 Playing；Qt frameSwapped 只能证明提交/交换，物理上屏时刻另测。

GameSessionVM 保留 running/foreground、resume/background/quit 及既有提示接口；新增 connectionState、stageVisible、errorText、budgetKind、hasRemainingTime。生产无额度时为 Unlimited 或 Unknown，不能用 0 秒触发退出；Finite 以单调时钟截止时间推进，默认低余额阈值仍为 1800 秒。二期以厂商额度更新为准。

| 状态/操作 | 视频音频 | 输入与会话 |
| --- | --- | --- |
| 连接中 | 加载层，允许底层完成首帧准备 | 可取消，远端输入关闭 |
| 前台播放 | 最新视频、音频 | 原始输入发远端，保留本地菜单快捷键 |
| 游戏菜单打开 | 视频继续、保留现有叠层 | releaseAll；菜单独占本地输入 |
| 返回桌面 | 持续消费码流/解码参考帧；不提交隐藏画面；默认静音 | 原连接保持，计时继续，本地桌面接收输入 |
| 继续 | 立即取最新可用画面，恢复音频 | 原连接恢复输入，不重新 launch |
| 退出/超时 | 停止媒体并清理缓冲 | 先释放远端输入，再断开；不关闭远端游戏 |

菜单关闭或恢复时，仍按住的键、按钮、偏离中位的摇杆先等待释放/回中，避免双投递与粘键。网络断开时本地记账清零，服务端清理注入状态；重连建立新的 transport generation，远端应用连续性和传输连接连续性分开记录。

TV/PC 形态切换会重启进程，与“返回桌面”不同。活动流切换前保存不含密钥的 ResumeDescriptor（hostId/appId/预算引用），释放输入、停止呈现并等待 GPU 引用，沿用现有 GL 停放门禁。停放失败不 spawn、不退出且恢复原呈现；成功后断开旧流释放 lease，再启动子进程，由子进程认证并 resume 原应用。启动失败父进程尝试恢复同一应用；不把失败误报成无缝恢复。密钥仍留受权限保护的配对存储。

一期时间与资源策略：连接超时10秒、传输连接后首帧超时5秒；重连按1/2/4秒退避（后续封顶4秒），总窗口30秒，取消立即终止。退出清理5秒仍未完成则显示失败并保留仍在使用的资源，不能强制释放/分离活线程或启动竞争的播放进程。压缩队列上限8个AU，溢出进入等待关键帧的解码epoch；关键帧请求限频250ms；显示候选容量2，PCM队列上限40ms。ResumeDescriptor单次消费、有效60秒，保存Linux bootId及CLOCK_BOOTTIME绝对截止点，同一boot内预算不重置，跨boot拒绝旧描述符。

### 6.3 媒体管线

所有接入层输出完整编码 access unit、codec、参数集/配置、时间戳、关键帧标识和缓冲所有权。SDK 回调返回后可能失效的内存必须在返回前复制或持有厂商允许的引用，不保存悬空指针。RTP 包、NAL 片段和完整帧不能混用。

MPP 独立解码线程；info-change 时重新分配/确认缓冲组；按 MPP/DPB 要求管理引用，不把整个解码池限制成两张。压缩队列有界，溢出或损坏时请求关键帧并重同步，不任意删除参考帧。显示候选采用容量 2、最新优先；GPU 使用中的帧保持引用直到同步完成，不能被解码器覆写。

首发视频路径为 MPP DMA-BUF → BSP 支持的 EGLImage 导入 → 自定义 Qt Quick OpenGL/GLES 节点/材质 → 单一显示管理者。校验 NV12 格式、stride、modifier、色彩范围/矩阵、同步和析构线程；不把 NV12 当普通 RGBA 纹理包装。目标无 CPU 逐帧 YUV→RGB 和 QImage 上传，但存在 GPU 合成开销。

实际 QPA、EGL 扩展和显示模式在板端探测后写入部署清单：已有 Wayland 时 compositor 唯一拥有 KMS；EGLFS 时由 Qt 平台集成拥有显示。不得自行在普通 QML Item 里竞争同一 CRTC。若 Qt 合成不能达到 240，进入统一显示管理者的 video/UI plane 优化评估；不能移除用户要求的叠层来宣布达标。

GameStageLayer 保持 overlayHost 与提示/菜单层，替换视频底层；fixture 模式仍可显示 PosterImage。PointerShield 与输入过滤协同，避免视频点击穿透桌面，同时不吞掉远端应收到的鼠标。

一期音频使用 common-c 的 Opus 回调、libopus、已有 SDL2 音频输出（48kHz 双声道基线），避免为此引入 QtMultimedia。播放队列有界；短时缺包用解码器补偿/静音，后台持续消费但静音，恢复不播放陈旧积压。SDK 若输出 PCM 则绕过解码器；输出其它编码必须显式实现/拒绝，不能假定均为 Opus。

### 6.4 原始输入与设置

现有 GamepadWorker 只向 UI 暴露方向语义，需增加 deviceId + 全量模拟轴/按钮原始快照，沿用唯一 SDL 事件线程；不另启竞争 SDL consumer。键盘使用物理键映射和明确的重复策略，排除 UI 合成事件。相对鼠标从实际平台获取未被屏幕边界截断的 delta；Wayland 使用其相对指针/约束能力，EGLFS 使用同一平台输入适配，不以光标坐标差假装无限相对移动。权限和能力由板端探测确定。

InputRouter 管 UI，StreamingInputRouter 管远端；InputOwnership 为 Local/Remote，切换前 releaseAll。一期一只活动手柄，保留现有设备切换规则，热拔插释放远端状态；多手柄扩展另议。

SettingsVM 增加 1440p 和 240fps，并区分请求档位、协商档位、实测吞吐与输出刷新率。用户请求超出能力时明确提示，不静默把 120 冒充 240。帧率/丢包 OSD 统计真实层级；UI 统计更新限频到 2Hz，不按 240Hz 更新 QML 文本。

## 7. 二期接口边界

VendorSdkBackend 实现与 MoonlightBackend 相同的会话/媒体契约；公网拥塞控制、ICE/TURN/DTLS/SRTP 由厂商 SDK 负责。客户端不再叠一套 WebRTC jitter buffer，也不自行假设 SDK 提供标准 libwebrtc VideoDecoderFactory。

厂商接入表必须确认：编码格式/profile/level、AU 或片段回调、时间戳单位/时钟域、关键帧请求、配置变更、buffer 生命周期、回调线程、音频格式、输入发送、断线恢复、额度更新、设备认证、实际 fps 上限。只有视频回调时只能完成视频接入，不能声称完整云游戏外设与音频已完成。

## 8. 验收与分阶段门禁

G0 板端：记录板卡型号/BSP/内核/MPP/Qt/QPA、显示接口、线材、EDID 和 1440p240 模式；离线 HEVC 硬解；带真实 Qt 菜单叠层呈现。未通过时明确目标未达，较低档仅用于继续软件联调。

G1 服务端：固定源码和依赖；UHD730、RTX30、RTX50 分别记录精确型号、驱动、显示、实际 encoder；标准 Moonlight 配对/播放/输入作为对照；严格编码选择和单会话竞争测试通过。

G2 客户端：状态机、取消/迟到回调、证书校验、码流恢复、缓冲生命周期、输入路由、预算和既有交互测试通过；无 GPU 测试不能替代 MPP/上屏验收。

G3 整机：动态帧号测试源验证真实不同帧，分别测捕获/编码/接收/解码/提交/物理呈现；静态画面重复帧不算真实 240。建议工程验收阈值：持续 30 分钟、每 60 秒窗口不同画面呈现率不低于目标的 99%、p99 帧间隔不大于 2 个刷新周期；随后 2 小时热稳态、100 次菜单/前后台、20 次断线恢复，资源不持续增长。阈值调整须留测量依据，不改“240”为低档成功。

输入到光子用高速摄影/光电测量，记录游戏渲染时间、显示器模式和 p50/p95/p99；一期建议目标 p95≤35ms、p99≤50ms，这是待首轮实测校准的工程目标，不是已验证能力。阶段耗时无法在没有跨机时钟校准时直接相减。

客户端继续执行 Qt 6.8.3、Debian12/GCC12.2、真实运行 qmllint 且输出零 Warning/Error、ctest、TV/PC offscreen 首帧；offscreen 仅验证启动，不验证视觉/ARM64/240。服务器使用上游 gtest、必要 Web 测试、实际 Windows 构建与设备验证。发布清单明确哪些档位在哪个硬件组合上通过。

## 9. 尚需实施阶段取证的项目

板卡具体型号/显示接线/BSP、真实 QPA 与 DMA-BUF 能力，通过 G0 实测锁定；厂商 SDK 细节在二期接入前锁定。它们不阻止仓库准备、纯逻辑开发和计划编制，但阻止对应性能/功能的通过声明。

## 10. 参考依据

- [Sunshine 开发基线](https://github.com/LizardByte/Sunshine/tree/d0b4a88ef727f8dabf80cc970a14739b55c885de)
- [Sunshine 构建说明](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2building.html)
- [Moonlight common-c](https://github.com/moonlight-stream/moonlight-common-c)
- [Rockchip MPP](https://github.com/rockchip-linux/mpp)
- [RK3576 官方规格](https://www.rock-chips.com/uploads/pdf/2024.3.18/191/RK3576%20Brief%20Datasheet.pdf)
- 客户端 AGENTS.md、docs/handover.md，以及本设计第 1 节固定基线的 QML/C++ 源码。
