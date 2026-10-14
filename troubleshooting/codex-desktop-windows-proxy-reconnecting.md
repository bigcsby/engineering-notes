# Windows Codex Desktop 反复 Reconnecting 1/5～5/5：系统代理正常但任务持续重连的排查与修复

> **已实际验证修复** · Windows / Codex Desktop / 本地 HTTP 代理 · 2026-10-14

## 1. 问题现象

在 Windows 使用 Codex Desktop 时，Windows 系统代理已开启，ChatGPT 网页也能访问，但 Codex 执行任务期间不断出现：

~~~text
Reconnecting 1/5
Reconnecting 2/5
...
Reconnecting 5/5
~~~

换成延迟较低的日本节点也无法解决。最终通过为 Codex **显式设置 HTTP_PROXY / HTTPS_PROXY**，在后续测试中消除了这一重连故障。

## 2. 环境

| 项目 | 实际配置 |
| --- | --- |
| 系统 | Windows |
| 应用 | Codex Desktop |
| 代理客户端 | 提供 HTTP/Mixed 端口的 Clash/Mihomo 类客户端 |
| 代理地址 | 127.0.0.1:7890 |
| 节点 | 日本 |
| Windows 系统代理 | 已开启 |

**7890 只是本机实际端口，其他用户需替换为自己的 HTTP/Mixed 代理端口。**

## 3. 排查过程

### 3.1 确认代理端口及基本连通性

一开始误用 7897，curl 返回 error 7（连接失败）。改为真实代理端口 7890：

~~~powershell
curl.exe -x http://127.0.0.1:7890 --max-time 15 -sS -o NUL -w "HTTP=%{http_code} TLS=%{time_appconnect}s TTFB=%{time_starttransfer}s TOTAL=%{time_total}s\n" https://chatgpt.com/
~~~

示例结果：

~~~text
HTTP=403 TLS=0.083483s TTFB=0.152128s TOTAL=0.152617s
~~~

对同一节点做 10 次测试，TTFB 中位数约为 156ms，但请求都返回 **HTTP 403**。

注意：403 说明收到拒绝响应，不意味着 ChatGPT 首页正常加载，更不能证明 Codex 模型接口/WebSocket 正常。它不是模型生成首个 Token 的耗时，也不能单凭这一状态码判断 Codex 重连原因。

### 3.2 Windows 系统代理设置正常

~~~powershell
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" | Select-Object ProxyEnable, ProxyServer, AutoConfigURL
netsh winhttp show proxy
~~~

输出要点：

~~~text
ProxyEnable = 1
ProxyServer = 127.0.0.1:7890
WinHTTP: Direct access (no proxy server)
~~~

这不矛盾：Windows 用户系统代理与 WinHTTP 并非同一套配置，WinHTTP 显示直连不一定是错误；不要因此直接全局更改 WinHTTP。

### 3.3 环境变量和 Codex .env 原先均未配置

~~~powershell
Get-ChildItem Env: | Where-Object { $_.Name -match 'PROXY' }
'HTTP_PROXY','HTTPS_PROXY','ALL_PROXY' | ForEach-Object { "$_ = $([Environment]::GetEnvironmentVariable($_, 'User'))" }
Test-Path "$HOME\.codex\.env"
~~~

实际结果：当前终端没有代理环境变量，用户级 HTTP_PROXY / HTTPS_PROXY / ALL_PROXY 为空，默认 Codex .env 不存在。

这并非直接证明错误，但提示相关网络客户端可能没有拿到**显式代理配置**。

### 3.4 Codex 的部分连接确实经过 7890

代理软件请求列表显示发往：

~~~text
tcp://chatgpt.com:443
~~~

此外，PowerShell 观察到 Codex 相关进程有到 127.0.0.1:7890 的 Established TCP 连接。

~~~powershell
$codexPids = @(Get-Process | Where-Object { $_.ProcessName -match 'codex|chatgpt' } | Select-Object -ExpandProperty Id)
Get-NetTCPConnection -ErrorAction SilentlyContinue | Where-Object { $codexPids -contains $_.OwningProcess -and $_.State -in @('Established','SynSent') } | Select-Object OwningProcess, RemoteAddress, RemotePort, State
~~~

因此**不是所有请求都没走代理**。这也不能证明流式请求或 WebSocket 必然正确继承了 Windows 系统代理。

## 4. 已验证的解决方案

在 Codex 环境文件中显式设置代理变量。

执行下列 PowerShell（**文件存在时不会覆盖已有配置**）：

~~~powershell
$codexHome = if ($env:CODEX_HOME) {
    $env:CODEX_HOME
} else {
    Join-Path $HOME ".codex"
}

$envFile = Join-Path $codexHome ".env"

if (Test-Path $envFile) {
    Write-Host "文件已存在，请先检查，不覆盖：$envFile"
} else {
    New-Item -ItemType Directory -Force -Path $codexHome | Out-Null

    @"
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
"@ | Set-Content -Path $envFile -Encoding ascii

    Write-Host "代理配置已创建：$envFile"
}
~~~

最终的环境文件内容：

~~~dotenv
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
~~~

- **HTTP_PROXY**：支持该变量的 HTTP 客户端所使用的代理。
- **HTTPS_PROXY**：支持该变量的 HTTPS 客户端所使用的代理。
- **NO_PROXY**：本地回环地址绕过代理。

文件已存在时，应在检查后**合并配置，而非覆盖**。代理端口需要换成你自己的真实端口。

**保存后完全退出 Codex Desktop（包括可能的后台进程），重新启动。**

### 结果

修改之前反复 Reconnecting 1/5～5/5；增加 Codex 独立代理变量并重启后，使用同一代理环境执行任务不再出现重连。未修改 WinHTTP，也没有依靠更换节点解决故障。

## 5. Root Cause 分析

最符合这次排障现象的解释是：**Windows 系统代理并不保证桌面应用内部每个网络组件都使用相同的代理解析机制**。

部分 HTTPS 请求可以正常使用系统代理，但其他流式通信或 WebSocket 组件可能使用不同代理配置，从而发生连接超时或重试。为 Codex 显式指定 HTTP_PROXY / HTTPS_PROXY 后，相关网络请求能够使用明确的本地代理地址。

这个判断与 OpenAI Codex 官方仓库的社区问题报告高度一致：

[openai/codex#29958 — Windows: WebSocket transport times out with respect_system_proxy, works with HTTP_PROXY/HTTPS_PROXY](https://github.com/openai/codex/issues/29958)

报告指出：在 Windows，系统代理下 HTTPS 可用但 WebSocket 超时并回退，设置 HTTP_PROXY / HTTPS_PROXY 后 WebSocket 正常。

**证据限制：** 本次没有抓取具体的 WebSocket 握手日志，因此不能证明 WebSocket 是唯一根因。可以确认的是显式代理配置与重连消失具有明确关联，现象与上游报告吻合。

## 6. 复验与回滚

建议用同一个节点先运行短任务，再运行需要长时间流式输出的真实任务，观察重连次数及代理客户端连接记录。

如果 .env 是此次**新建且仅包含上述配置**的文件，可以回滚：

~~~powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
Remove-Item (Join-Path $codexHome ".env")
~~~

若原本就有 .env 文件，只撤销本次新增行，不要删除整个文件。

之后如果代理软件更改监听端口，要同步修改 .env；只切换远端节点而不改变本地端口，通常不用修改。

## 7. 工程经验

**系统代理正确 ≠ 应用内部所有网络组件都正确使用系统代理。**

建议的排查顺序：

1. 确认本地代理端口；
2. 验证目标 HTTPS 请求的连通性与状态码；
3. 检查 Windows 系统代理和用户环境变量；
4. 观察应用进程的实际 TCP 连接；
5. 必要时检查 WebSocket / 流式连接日志；
6. 进行可撤销的显式代理实验；
7. 用真实任务验证，不只看机场 Ping 和 curl TTFB。

---

## 参考

- [OpenAI Codex GitHub](https://github.com/openai/codex)
- [openai/codex Issue #29958](https://github.com/openai/codex/issues/29958)

*本文记录一次真实验证有效的修复，不保证适用于所有 Codex Desktop 版本或所有 Reconnecting 故障。*
