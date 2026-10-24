# 关闭 Microsoft Defender：Windows 11 26H2

资料核对：2026-10-24。适用于个人 Windows 11 电脑；公司或学校管理的设备以组织策略为准。

微软已宣布 26H2 正式发布。下面依据当前 Windows 11 与 Defender 官方文档整理，并非在你的电脑上实际测试过的结果。[版本信息](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-26H2)

## 先选需要的方式

| 目的 | 方式 | 效果 |
| --- | --- | --- |
| 临时安装或排查软件 | 关闭实时保护 | 通常会自动恢复，计划扫描仍可运行 |
| 调整实时保护策略 | 本地组策略 | 受篡改防护、组织策略和系统更新影响 |
| 不安排常规扫描 | 修改扫描计划 | 不等于关闭实时保护或手动扫描 |
| 改用其他杀毒软件 | 安装能注册到 Windows 安全中心的杀毒软件 | Defender 的运行模式由系统自动调整 |

关闭保护后，文件检查能力会减少。本文不提供删除服务、驱动或系统组件的步骤。

## 1. 临时关闭实时保护

打开 **Windows 安全中心 → 病毒和威胁防护 → 管理设置 → 实时保护**，将开关关闭。

这是临时设置，系统通常会在一段时间后重新开启；关闭实时保护也不会停止所有计划扫描。[微软说明](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-security-center-antivirus)

## 2. 使用本地组策略

适用于带本地组策略编辑器的版本，如专业版、企业版、教育版；家庭版通常没有 `gpedit.msc`。

### 关闭篡改防护

在 **Windows 安全中心 → 病毒和威胁防护 → 管理设置** 中关闭“篡改防护”。

管理员 PowerShell 检查：

```powershell
Get-MpComputerStatus | Select-Object IsTamperProtected
```

`False` 表示已关闭。篡改防护开启时，某些修改可能看似成功，实际被忽略；正常运行状态下不能用普通 PowerShell 命令关闭篡改防护。受组织管理时，需由管理端处理。[篡改防护说明](https://learn.microsoft.com/en-us/defender-endpoint/tamper-protection-windows-configure)

### 设置策略

按 `Win + R`，输入 `gpedit.msc`，进入：

**计算机配置 → 管理模板 → Windows 组件 → Microsoft Defender 防病毒 → 实时保护**。

| 策略 | 设置 |
| --- | --- |
| 关闭实时保护 | 已启用 |

这里“已启用”表示启用“关闭实时保护”这条策略。不要将其理解为开启实时保护。[组策略说明](https://learn.microsoft.com/en-us/defender-endpoint/configure-real-time-protection-microsoft-defender-antivirus)

管理员 PowerShell 刷新策略，随后重启检查：

```powershell
gpupdate /force
```

如果只是临时排查，也可在管理员 PowerShell 中请求关闭实时监控，但同样受篡改防护和管理策略限制：

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

这不等于卸载 Defender，也不保证重启或更新后永远保持关闭。[参数说明](https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference)

## 3. 关闭计划扫描（可选）

先查看并记下原值，便于恢复：

```powershell
Get-MpPreference | Select-Object ScanScheduleDay, RemediationScheduleDay
```

管理员 PowerShell 设置：

```powershell
Set-MpPreference -ScanScheduleDay 8 -RemediationScheduleDay 8
```

两项参数的 `8` 均表示 `Never`。这只调整对应计划，不会关闭实时保护，也不能阻止手动扫描或其他管理工具发起的扫描。[扫描计划参数](https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference)

如需检查系统计划任务：

```powershell
Get-ScheduledTask -TaskPath "\Microsoft\Windows\Windows Defender\" |
    Select-Object TaskName, State
```

不必将清理、校验等维护任务全部禁用，也不能仅凭任务列表判断 Defender 已完全停止。

## 4. 验证是否生效

重启后，在 PowerShell 中执行：

```powershell
Get-MpComputerStatus | Select-Object AMRunningMode, AMProductVersion, AntivirusEnabled, RealTimeProtectionEnabled, BehaviorMonitorEnabled, IoavProtectionEnabled, IsTamperProtected

Get-MpPreference | Select-Object DisableRealtimeMonitoring, ScanScheduleDay, RemediationScheduleDay
```

| 项目 | 如何理解 |
| --- | --- |
| `RealTimeProtectionEnabled = False` | 实时保护当前关闭 |
| `DisableRealtimeMonitoring = True` | 配置为禁用实时监控，仍需核对实际状态 |
| `AMRunningMode = Normal` | Defender 处于活动模式，不能写成“完全停用” |
| `AMRunningMode = Passive` | 被动模式，不是卸载；部分能力仍可能工作 |
| `ScanScheduleDay = 8` | 常规扫描计划为 Never |

`AMProductVersion` 是 Defender 平台版本。实际行为还受平台更新影响，不能只看 Windows 的“26H2”标签。[运行模式说明](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-compatibility)

原笔记中的 `1000 / 1001 / 1002` 扫描日志可以辅助排查；查不到这些事件，不能证明所有实时检查均已停止。

## 5. 想让其他杀毒软件接管

安装兼容且正常运行、能注册到 Windows 安全中心的杀毒软件后，系统会自动调整 Defender 状态。普通个人电脑可能进入禁用模式；启用智能应用控制或接入 Defender for Endpoint 等情况下，也可能进入被动模式。

在 **Windows 安全中心 → 病毒和威胁防护 → 管理提供程序** 中确认当前杀毒软件。其他杀毒软件失效或卸载后，Defender 可能自动恢复。[兼容性说明](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-compatibility)

## 6. 恢复保护

1. 将前面修改的“关闭实时保护”策略恢复为“未配置”，执行 `gpupdate /force`。
2. 在 Windows 安全中心重新开启实时保护和篡改防护。
3. 如果使用过 PowerShell 禁用实时监控，恢复命令如下；若被篡改防护拦截，先在界面关闭篡改防护，完成后再开启。

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
```

4. 扫描计划恢复为第 3 节记下的原值，不要把 `8` 当作所有电脑都应采用的默认值。
5. 重启，再检查 `RealTimeProtectionEnabled`。如果另一款杀毒软件正在接管，Defender 的状态仍由系统协调。

## 不再沿用的旧做法

- 不把 `DisableAntiSpyware`、`DisableAntivirus` 注册表项当作新版系统的可靠关闭方案；微软已将这些旧设置淘汰。[官方说明](https://learn.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/security-malware-windows-defender-disableantispyware)
- 不把“安全中心窗口关闭”“计划任务被禁用”或“日志为空”当作完全关闭的证据。
- 不声称已实现“26H2 永久关闭且更新后不会恢复”；本次查阅的微软资料没有给出这种保证。
