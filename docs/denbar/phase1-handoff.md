# denbar-stream 一期 Windows 服务端开发交接

日期：2026-10-27，Asia/Shanghai。用户已批准修订2的两端集成设计与一期计划，并明确在本机新开Agent实施服务端；另一台机器实施客户端。本说明是已批准计划的执行交接，不新增产品范围。

## 1. 仓库和已发布基线

- 本机目录：`E:/VisualStudioProjects/denbar-stream`。
- origin：`https://github.com/TangChengLin/denbar-stream.git`。
- upstream：`https://github.com/LizardByte/Sunshine.git`。
- 开发分支：`codex/denbar-stream-integration`。继续该分支，保护已有改动；如接收时已有人开发，先核对实际HEAD和工作区，不能重置到本文旧提交。
- Sunshine源码开发基线：`d0b4a88ef727f8dabf80cc970a14739b55c885de`；设计/计划修订2提交：`89a2e52fa15df5904916a9a965841900eda64658`，已推送并验证与远程一致。本交接说明作为其后续提交发布。
- 目前已完成Fork、完整克隆、upstream配置和文档准备；尚未实施S1—S6产品任务。已初始化的47个递归子模块属于准备基线，接手时仍需检查状态。

已批准的设计和计划位于：

1. `docs/superpowers/specs/2026-10-27-denbar-stream-integration-design.md`
2. `docs/superpowers/plans/2026-10-27-denbar-stream-server-plan.md`

仓内交接入口为 `docs/denbar/phase1-handoff.md`。新Agent先读 `AGENTS.md` 和以上文档，再核对当前源码与工具链。不要从移动的upstream/master重新开始，也不要因目录叫VisualStudioProjects就改成未经验证的MSVC工程。

## 2. 固定决策与实施顺序

服务端为C++23 + Sunshine轻量Fork，复用Windows捕获、Quick Sync/NVENC、音频、配对、加密、RTSP/RTP/FEC和输入注入。Denbar新增集中在profile、严格编码选择、诊断、单会话admission与部署验证，保持上游易合并。客户端C++17/Qt6.8.3不约束服务器第三方构建标准。

按服务器计划实施：

1. S1：探测本机实际GPU/驱动/显示与MSYS2 UCRT64工具链，固定依赖，构建上游基线和现有测试；用标准桌面Moonlight完成1080p60图像、音频、键鼠/手柄对照。客户端或第二种GPU未到位不阻塞本机能完成的工作。
2. S2/S3：profile schema、允许媒体组合、配置事务与能力导出；核对实际编码器，严格模式不能将软件或其它编码器回退报成目标硬编成功。
3. S4：单活动流、Reserved/Active/Configuring互斥、旧token保护、失败释放，以及launch与RTSP完整参数校验；与S2事务接口闭环。
4. S5：安装冲突检测、升级、配对与配置保留、回退验证；按计划复用上游机制。
5. S6：UHD730、具体RTX30/RTX50及客户端联合验收；无实机的组合标待测，不能由某一GPU结果外推。

S1不是整个任务的终点。先交付可复现的构建与对照结果，再按依赖继续能开展的任务；确实受外部设备或客户端进度阻塞时，记录缺失条件并推进独立工作。

一台服务器只允许一个活动串流连接；退出本地流不关闭远端游戏。HEVC Main、8-bit、4:2:0、SDR为共同高帧率候选；目标为2560×1440、240个不同画面/秒和240Hz输出，是否通过由动态帧号及真机测量证明。标准客户端、板端moonlight-embedded与最终denbar-shell的证据分别记录。

## 3. 构建、验证与提交

严格执行服务器 `AGENTS.md`：Windows使用MSYS2 UCRT64，构建目录以 `cmake-build-` 开头；测试使用test_sunshine/gtest公共头规则，源码行为变更增加适用回归，维护Doxygen与clang-format。工具链和依赖准备属于S1，不以本机已有Qt Creator或Visual Studio代替核验。

按计划记录命令、退出码、失败原因、依赖清单和实际设备。产品开发构建/测试与Release性能验收分开。没有测试或硬件证据的步骤不勾选完成；不为通过而禁用测试或静默降档。

按任务本地提交，避免修改无关上游文件。前一次授权的远程推送已完成；本交接默认后续实现先做本地提交，远程发布、合并master、安装替换现有服务或对外部署按用户后续明确指示执行。普通构建、测试及隔离的本机联调按已批准计划推进。

## 4. 与另一台机器的客户端协作

客户端仓库为denbar-shell，生产common-c + MPP，Qt合成/DRM双图层由客户端C0b取证选型；服务端不接管其呈现实现，也不在本仓库添加客户端代码。

服务端阶段交付：

- S1：实际主机地址/端口/网络可达条件、配对流程与测试窗口、服务器commit/依赖指纹、GPU/驱动/输出与实际编码器、标准Moonlight对照结果和测试码流参数。
- S2/S4：无密钥host-profile.json、明确的允许档位、Experimental/Verified状态，以及参数冲突、busy、恢复与退出语义。
- S6：与客户端共用runId，关联请求/协商/捕获/编码和客户端接收/解码/提交/实际呈现；保留帧号、时延、热稳态和异常恢复证据。

客户端返回BSP/MPP/Qt/QPA/显示指纹、两种呈现路线比较、选定基线和C9验收报告。双方交换资料和测试时间，不需要共享私钥或服务器管理密码。共享协议/配置字段如需调整，先同步两仓库共同设计，不能各自修改接口后再猜测兼容性。

## 5. 可直接粘贴给本机服务端Agent的启动文案

```text
请在本机开始 denbar-stream 一期 Windows 串流服务端开发，并按已批准计划持续推进。

项目目录：E:\VisualStudioProjects\denbar-stream
当前开发分支：codex/denbar-stream-integration
origin：https://github.com/TangChengLin/denbar-stream.git
upstream：https://github.com/LizardByte/Sunshine.git
已发布设计/计划基线：89a2e52fa15df5904916a9a965841900eda64658；交接文档可能有后续提交，接手时核对实际HEAD，保护已有改动。

先读：
1. AGENTS.md
2. docs/denbar/phase1-handoff.md
3. docs/superpowers/specs/2026-10-27-denbar-stream-integration-design.md
4. docs/superpowers/plans/2026-10-27-denbar-stream-server-plan.md

我已批准修订2设计、实施计划和本次服务端实施。由当前主代理逐任务执行，必要时辅助调研与独立审查；若具备superpowers技能，使用superpowers:executing-plans。无需重新讨论技术选型或重复请求开始实施的确认。

本任务仅负责服务端，客户端由另一台机器单独开发。保持C++23 + Sunshine轻量Fork，按AGENTS使用MSYS2 UCRT64、CMake/Ninja，保持固定子模块和依赖，复用上游采集、编码、音频、协议与输入。不要修改Moonlight线协议，也不要自动追逐upstream/master。

从S1开始：探测本机GPU/驱动/显示和工具链，建立依赖与构建基线，运行现有测试，并用标准Moonlight跑通1080p60对照。然后按依赖推进S2/S3、S4、S5、S6；不要只提交一份计划或停在环境说明。重点落实profile权威、实际硬编选择、单活动流与失败释放、恢复语义和部署回退。

没有其它GPU、RK3576或客户端联调条件时，记录待测并继续独立工作；不能声称全部GPU或1440p240已通过。生产性能用Release和真实动态帧号验证，不以请求参数或重复画面冒充吞吐。

按任务测试、本地提交并更新进度，报告实际命令与结果、硬件待测项、下一步。向客户端提供S1联调资料、S2/S4的host-profile.json和S6统一runId报告。后续推送、合并master或替换现有服务部署，等待我另行指示；普通构建、测试及隔离联调直接推进。
```
