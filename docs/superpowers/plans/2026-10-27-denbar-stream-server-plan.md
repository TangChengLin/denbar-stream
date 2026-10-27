# Denbar Windows 服务端一期 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Sunshine 轻量 Fork 上交付可诊断、可更新、严格硬编选择且单活动流的 Windows LAN 服务端，与标准 Moonlight 及 denbar-shell 联通。

**Architecture:** 复用 Sunshine 捕获、编码、协议、音频和输入。Denbar 只增加 profile、admission 与诊断模块，配置复用原有 Web 管理认证及现有部署方式。发布资格由真实设备测试决定。

**Tech Stack:** C++23、MSYS2 UCRT64、CMake/Ninja、上游 Qt/Vue、gtest；Quick Sync/NVENC。

**Spec:** `../specs/2026-10-27-denbar-stream-integration-design.md`。输出目录中的副本与设计文件同目录；仓内版本使用此相对路径。

## Global Constraints

- 工作目录 `E:/VisualStudioProjects/denbar-stream`，开发基线 `d0b4a88ef727f8dabf80cc970a14739b55c885de`，分支 `codex/denbar-stream-integration`。
- 主 C++23，保留上游第三方/CUDA 标准；Windows 使用 MSYS2 UCRT64；构建目录 `cmake-build-*`。
- 一台主机同一时刻只允许一个活动串流连接；配对/加密/RTSP/RTP/FEC 线协议不修改。
- 一期 HEVC Main、8-bit、4:2:0、SDR；1440p240 只有真实验证通过才标为已验证。
- capture 基线 `ddx`；encoder 为 `quicksync` 或 `nvenc`；硬件失败不得静默软件回退。
- 遵循 AGENTS.md：测试包含 tests_common.h、修改方法有回归、Doxygen 与 clang-format；构建配置变更只做配置/构建检查，不编造测试工程。
- 未经设备验证不得发布性能结论；本计划未执行的步骤保持未勾选。

## Review Focus

1. 指定编码器失败后上游自动 fallback，必须明确失败且保留实际探测信息——S3。
2. 两个 launch/resume 与 profile 应用同时进入，不能双占或边串流边改配置——S2/S4。
3. 客户端能力清单过期、尺寸合法但组合不合法，必须拒绝而不是悄悄覆盖——S2/S4。
4. pending 超时/取消后的旧回调不得释放新连接 lease——S4。
5. 升级/重启失败不能毁掉配对、用户配置或原安装——S5/S6。

## 文件布局与类型契约

新增 profile/session_policy/diagnostics 位于 `src/denbar/`，加入现有 `cmake/compile_definitions/common.cmake` 源清单；对应测试放 `tests/unit/test_denbar_*.cpp`。前端扩展独立为 `src_assets/common/assets/web/DenbarProfiles.vue` 并通过 `router.js` 接入。没有真实改动需求的采集、编码实现保持上游。

拟新增类型位于 `namespace denbar`，采用上游 snake_case 风格：

- `mode_request_t`：codec(string)、width/height/fps/bitrate_kbps(int)。
- `allowed_mode_t`：codec、width/height/fps/max_bitrate_kbps、qualification(Experimental/Verified)、qualification_id(string)。
- `profile_t`：schema_version=1、id/revision/capture/encoder/adapter_name/output_id(string)、require_hardware/allow_experimental(bool)、allowed_modes(vector)。
- `selection_result_t`：requested_encoder/actual_encoder、capture_adapter/encode_adapter/output_id、hardware(bool)、error(string)。
- `lease_token_t`：单调递增 uint64_t；旧 token 不得操作新 lease。
- `diagnostic_snapshot_t`：schema_version、profile_revision、selection_result_t、请求/实际媒体参数、阶段时间、错误；不包含证书私钥、PIN 或管理密码。

现有 Sunshine 类型通过小范围 adapter 转换到这些类型；不会在跨项目边界传 C++ 对象。

## 验证命令

所有 Windows 原生构建/测试使用仓库规定前缀。下列命令从仓库根运行；先按官方 docs/building.md 准备依赖，记录版本，不以已安装 Qt Creator/Visual Studio 代替工具链核验。

```powershell
& 'C:\msys64\msys2_shell.cmd' -defterm -here -no-start -ucrt64 -c 'cmake -S . -B cmake-build-denbar -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_TESTS=ON'
& 'C:\msys64\msys2_shell.cmd' -defterm -here -no-start -ucrt64 -c 'cmake --build cmake-build-denbar'
& 'C:\msys64\msys2_shell.cmd' -defterm -here -no-start -ucrt64 -c './cmake-build-denbar/tests/test_sunshine.exe --gtest_color=yes'
```

定向测试在最后一条追加 `--gtest_filter=DenbarProfile.*` 等本文测试组。先观察新增用例因缺少行为失败，再实现，再全组通过；不把编译错误当唯一行为验证。测试构建带覆盖率开销，性能验收另用 Release、BUILD_TESTS=OFF 的 `cmake-build-denbar-release`。

## S1 固定可复现基线并完成标准客户端对照

**Files:** 新增 `docs/denbar/build-baseline.md`、`docs/denbar/validation/host-inventory.json`；必要时修改 `cmake/prep/options.cmake` 的发布者配置入口，不改语言标准。

**Interfaces:** 输出固定源码/全部子模块/工具链清单、服务启动方式、首台主机实际捕获与编码结果；后续任务沿用这份基线。

- [ ] 核对 master 与开发分支、子模块无未初始化/偏离；清单包含 Windows 版本、GPU精确型号、驱动、显示模式、输入驱动和上游提交。
- [ ] 按官方工具链配置、构建、运行现有 test_sunshine；保存命令、退出码及失败详情。上游既有失败单独定位，不能通过关闭测试假装成功。
- [ ] 用原有配置界面完成凭据、配对及 Desktop 应用确认；标准 Moonlight 跑通 1080p60 图像、音频与输入，记录实际编码器。
- [ ] 以 `docs: record Denbar server baseline` 提交可审查的配置说明；不得提交密码、证书、设备个人信息或构建产物。

## S2 实现 profile schema、配置事务与能力导出

**Files:** 新增 `src/denbar/profile.h/.cpp`、`tests/unit/test_denbar_profile.cpp`；修改 `src/confighttp.cpp`、`cmake/compile_definitions/common.cmake`。新增 `src_assets/common/assets/web/DenbarProfiles.vue`、`tests/web/DenbarProfiles.test.js`；修改现有 `router.js`。

**Interfaces:** `parse_profile(std::string_view)->std::expected<profile_t,std::string>`；`validate_request(const profile_t&,const mode_request_t&)->std::expected<void,std::string>`；`export_host_profile(const profile_t&,std::string_view host_id)->std::string`。配置应用通过 S4 的 idle-only 事务 token，不直接改运行中的 config::video。

- [ ] 增加 DenbarProfile 测试：未知schema/字段、负码率、重复id、非法encoder、单独合法但组合错误、超过码率上限、管理apply的expectedRevision冲突、合法HEVC1440p240组合、Experimental默认拒绝/验收环境显式允许且不标Verified。断言失败无配置文件写入。
- [ ] 运行 `--gtest_filter=DenbarProfile.*` 观察行为失败，再用现有 JSON 依赖实现解析/验证；不新增第二套配置框架。
- [ ] 在原认证/CSRF保护内增加 `GET /api/denbar/profiles`、`POST /api/denbar/profiles/apply` 和无密钥能力清单下载。它们是新增管理API，不能假称上游原有；运行中返回 conflict，认证失败不写文件。
- [ ] 增加临时目录事务测试：写入/重命名失败保留旧配置、配对文件字节不变；S4 token尚未落地时用接口fake，禁止临时绕过idle判定。
- [ ] 增加 Web 用例：展示requested与verified、busy时禁止应用、过期能力提示、重启失败保留错误；`npm test -- tests/web/DenbarProfiles.test.js` 通过后跑相关管理回归。
- [ ] 提交 `feat(denbar): add validated host profiles`。

## S3 严格编码选择和实际结果诊断

**Files:** 新增 `src/denbar/diagnostics.h/.cpp`、`tests/unit/test_denbar_diagnostics.cpp`；修改 `src/video.h/.cpp`、`src/config.h/.cpp`、`src/confighttp.cpp`，复用现有 `test_video.cpp`。

**Interfaces:** `validate_selection(const profile_t&,const selection_result_t&)->std::expected<void,std::string>`；`serialize_diagnostics(const diagnostic_snapshot_t&)->std::string`；在实际 chosen_encoder 决定处填充 selection_result_t。Denbar专有严格选择配置默认只对 Denbar profile 启用，保持原上游行为可辨认。

- [ ] 增加测试：请求nvenc却得到software/quicksync时严格模式失败；宽松上游模式保留原行为；实际设备与输出信息不被请求值覆盖；序列化不含秘密。
- [ ] 定向运行 DenbarDiagnostics 与相关 Video 用例，确认回退误报场景先失败。
- [ ] 在 probe_encoders 的结果接缝实现校验及日志，沿用现有NVENC/QSV代码；同名适配器歧义明确报错。新增API和方法补Doxygen。
- [ ] 实际关闭/禁用目标编码能力的可恢复测试环境中验证启动失败提示，恢复后可重试；仅mock不能标为硬件通过。
- [ ] 定向测试、现有视频测试及格式检查通过；检查格式工具未引入无关diff；提交 `feat(denbar): verify actual hardware encoder selection`。

## S4 单活动连接与两阶段会话参数校验

**Files:** 新增 `src/denbar/session_policy.h/.cpp`、`tests/unit/test_denbar_session_policy.cpp`；修改 `src/nvhttp.cpp`、`src/rtsp.h/.cpp` 及必要的 `src/stream.cpp` 生命周期接缝，扩展当前已有 `tests/unit/test_rtsp.cpp`。

**Interfaces:** `admission_gate::reserve(std::string client_id)->std::expected<lease_token_t,std::string>`；`activate(lease_token_t)->bool`；`release(lease_token_t)->void`；`begin_config_update()->std::expected<lease_token_t,std::string>`；`end_config_update(lease_token_t)->void`。状态 Idle/Reserved/Active/Configuring 受同一锁保护；Reserved 超时继承上游握手计时机制，超时事件携token。

- [ ] 增加 DenbarSessionPolicy 测试：并发reserve仅一个成功；config与launch互斥；旧token释放新会话无效；重复release幂等；超时/编码失败后可再reserve。
- [ ] 增加launch/resume→ANNOUNCE集成用例：只填部分媒体字段的launch不能绕过最终RTSP完整组合校验；不匹配释放lease且不激活流。
- [ ] 定向失败证据后实现gate；lease进入现有launch_session_t/会话对象的生命周期，所有失败路径统一RAII/幂等释放，不依赖只有session_count的检查。
- [ ] 接通S2配置事务；应用前不让新launch穿过。网络断连释放传输lease，远端应用保留；busy恢复请求不取消他人应用。
- [ ] 两台标准客户端并发开始/恢复、pending超时重试、profile同时应用的实机流程通过；提交 `feat(denbar): enforce single streaming admission`。

## S5 部署、升级与回退

**Files:** 新增 `docs/denbar/deployment.md`、`tools/denbar/verify-install.ps1`；按实际需要修改现有 `cmake/packaging/windows*.cmake`、`tools/sunshinesvc.cpp`、`src_assets/windows/misc/service/` 和防火墙脚本，避免无依据全改服务名。

**Interfaces:** 验证脚本输出机器可读报告，字段为packageRevision/serviceStatus/portConflict/inputDriverStatus/configBackupPath/errors；退出0仅表示安装验证通过，不表示240通过。

- [ ] 定义安装前已有Sunshine服务/端口/配置的检测，冲突明确退出且不改原服务。制定单实例安装步骤和凭据备份权限。
- [ ] 实现最小包装与发布者信息，复用上游服务机制；驱动安装和版本按当前依赖实际验证，不假设旧ViGEm流程适用。
- [ ] 在测试机执行全新安装、重复安装、失败回退、升级后配对保留、卸载不删除用户保留配置；记录实际输出。脚本若改变用户状态，先使用测试目录/测试机覆盖失败路径。
- [ ] 运行需要的打包检查，生成哈希清单；提交 `build(denbar): document and verify Windows deployment`。

## S6 三类GPU和两端整机验收

**Files:** 新增 `docs/denbar/validation/lan-matrix.md`、`docs/denbar/validation/result.schema.json`、`tools/denbar/collect-session-report.ps1`；诊断所需采样扩展 `src/denbar/diagnostics.*` 及对应测试。

**Interfaces:** 与客户端计划 C9 使用同一 runId；报告含硬件/源码/配置指纹、请求和实际参数、distinctFrames/duplicates/drops、阶段耗时及采样定义。p95/p99需明确样本集合，不能从min/max/avg推算。

- [ ] 测试统计窗口、重复帧不计新帧、乱序/时钟回拨不生成负延迟；实现报告与客户端可关联的标识。
- [ ] 分别验证UHD730、具体RTX30型号、具体RTX50型号；每种记录标准Moonlight与denbar-shell结果，硬件未到列待测。
- [ ] 按设计G3执行30分钟动态帧号、2小时热稳态、100次前后台和20次断线；带实际Qt叠层测240与输入到光子，禁用测试覆盖率开销。
- [ ] 仅把匹配硬件指纹且通过的组合加入allowedModes已验证清单；失败保留证据，低档通过不能代替240。
- [ ] 固定发布commit/tag和依赖、报告、回退版本；提交 `docs: record Denbar LAN qualification`。

## 实施顺序与自查

S1→S2/S3→S4→S5→S6；S2的配置提交依赖S4事务接口，可先用fake完成纯逻辑，发布前必须联调。客户端C0可与S1并行；S1提供标准客户端对照后客户端C3开始真实联调；S6依赖客户端C9。

覆盖检查：profile权威(S2/S4)、硬件选择(S3)、单会话及失败释放(S4)、原生采集/协议复用(S1)、部署回退(S5)、实际240与上游同步发布(S6)。接口名称和类型以上述契约为准；新增需求先更新设计，避免在任务中偷换协议或目标。
