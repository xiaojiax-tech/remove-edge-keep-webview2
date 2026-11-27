# 卸载 Edge（保留 WebView2）操作手册

> 来源：https://rbq.dev/posts/2025-03-17-uninstall-edge/ （Remove-MS-Edge 方案）
> 整理时间：2026-11-24
> 适用：Windows 11（其他版本流程可能相同）
> 风险：删除系统组件，不可逆操作较多，无法保证零风险。请先创建系统还原点/备份，严格按顺序执行，并先做完 Windows 更新。
> 免责声明：本文档非微软官方文档。第三方卸载工具、Defender 排除项、删除 WebView2 旧目录均有风险，请自行校验下载文件并确认命令含义。

---

## 0. 这份手册解决了什么

原文所称的 `Remove-Edge.exe` “只卸载 Edge，保留 WebView2”，**实测不成立**：

- 它会删除 `edgeupdate` / `edgeupdatem` 服务和整个 `HKLM\...\EdgeUpdate` 注册表树
- WebView2 的文件还在，但**登记信息被删**，之后新启动的 WebView2 程序会报“运行时未安装”
- 因此必须在本手册第 2 步用官方安装包把 WebView2 修复回来

按本手册执行后：Edge 彻底移除，WebView2 正常运行且可继续更新。

---

## 1. 变更清单（执行完会改什么）

| # | 变更 | 方式 | 可否回滚 |
|---|---|---|---|
| 1 | 卸载 Edge：程序目录、EdgeCore、快捷方式、计划任务、`microsoft-edge:` 协议、AppX Stable | Remove-Edge.exe | 可重装 |
| 2 | 修复 WebView2 登记 + 恢复更新服务（版本可能升级） | 微软 Evergreen 安装包 | 可重装 |
| 3 | 删除旧 WebView2 运行时目录 | 重启后管理员手动删除 | 无需回滚 |
| 4 | 删除残留自启项 `MicrosoftEdgeAutoLaunch_*` | 注册表 | 可恢复 |
| 5 | 卸载残包 `Microsoft.MicrosoftEdgeDevToolsClient` | Remove-AppxPackage | 商店重装 |

**不会动的东西**：`Microsoft.Win32WebViewHost`（系统组件）、`C:\Program Files (x86)\Microsoft\EdgeWebView`、Edge 更新器服务（修复后它们是 WebView2 用的）。

---

## 2. 前置条件

1. **先完成 Windows Update**。原文明确警告：系统没更完就卸 Edge，可能进入“更新失败 → 回滚 → 再失败”的循环。若已发生，解决方法是装回 Edge、更完系统、再卸一次。
2. 把默认浏览器改成非 Edge（例如 Chrome），否则卸载后网页/链接打不开。
3. 准备好一个**管理员 PowerShell**：开始菜单右键 →「终端(管理员)」。
4. 确认 `github.com` 是否可达；不可达也没关系，下面走 `api.github.com`。
5. 建议先创建系统还原点，并确认重要数据已备份。

---

## 3. 操作步骤

### 第 1 步 · 卸载 Edge【管理员终端】

第一段：下载并校验哈希。成功会输出 `HASH OK`，并暂时保留 Defender 排除项。

```powershell
$d="$env:TEMP\edge-uninstall-tool"
New-Item -ItemType Directory -Force $d|Out-Null
Add-MpPreference -ExclusionPath $d

try {
  $r=Invoke-RestMethod 'https://api.github.com/repos/ShadowWhisperer/Remove-MS-Edge/releases/latest'
  $a=$r.assets|Where-Object Name -eq 'Remove-Edge.exe'
  if (-not $a.digest) { throw 'release asset digest 缺失，停止' }
  $a.digest

  $h="Accept: application/octet-stream"
  $w="User-Agent: curl/8"
  & curl.exe -L --retry 2 -H $h -H $w -o "$d\Remove-Edge.exe" $a.url
  if ($LASTEXITCODE -ne 0) { throw "curl 下载失败: $LASTEXITCODE" }

  $sum=(Get-FileHash "$d\Remove-Edge.exe" -Algorithm SHA256).Hash
  if ($a.digest -eq "sha256:$($sum.ToLower())") {
    'HASH OK'
  } else {
    throw "HASH MISMATCH: $sum"
  }
} catch {
  Remove-MpPreference -ExclusionPath $d -EA 0
  throw
}
```

确认输出 `HASH OK` 后，执行第二段。执行完会立即撤销 Defender 排除项：

```powershell
$d="$env:TEMP\edge-uninstall-tool"
try {
  Start-Process "$d\Remove-Edge.exe" -Wait
} finally {
  Remove-MpPreference -ExclusionPath $d -EA 0
}
```

说明：

- **必须先加 Defender 排除**。实测 Defender 会把该文件判为 `Trojan:Win32/Yomal!rfn` 直接删除（PyInstaller 单文件包的常见误报），不加排除文件根本留不住。
- 第二段会静默卡几分钟（无窗口、无提示），等 PowerShell 提示符回来即可。
- 如果第一段 `HASH OK` 后没有继续执行第二段，请手动撤销排除项：`Remove-MpPreference -ExclusionPath "$env:TEMP\edge-uninstall-tool" -EA 0`。
- 如果 `curl` 下载超时，说明 `api.github.com` 也不通，需要先解决网络。
- 第三方卸载工具本身有风险；执行前请确认哈希与 release 一致，并自行承担风险。

### 第 2 步 · 修复 WebView2【管理员终端】

```powershell
$d="$env:TEMP\webview2-repair"
New-Item -ItemType Directory -Force $d|Out-Null
& curl.exe -L "https://go.microsoft.com/fwlink/?linkid=2124703" -o "$d\MicrosoftEdgeWebview2Setup.exe"
if ($LASTEXITCODE -ne 0) { throw "下载 WebView2 安装包失败: $LASTEXITCODE" }

$sig=Get-AuthenticodeSignature "$d\MicrosoftEdgeWebview2Setup.exe"
$sig | Format-List Status,SignerCertificate
if ($sig.Status -ne 'Valid' -or -not $sig.SignerCertificate -or $sig.SignerCertificate.Subject -notmatch 'Microsoft Corporation') {
  throw 'WebView2 安装包签名校验失败，停止'
}

Start-Process "$d\MicrosoftEdgeWebview2Setup.exe" -ArgumentList '/silent','/install' -Verb RunAs -Wait
```

- 签名必须是 `Valid` + `Microsoft Corporation`。
- 该安装包会把 WebView2 升到最新版并重建登记，**不会装回 Edge**。
- 示例版本变化：157.x → 158.x（以实际安装包为准）。

### 第 3 步 · 清理残留

#### 3.1 清 Edge 自启残留【普通权限即可】

```powershell
$k='HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'
(Get-ItemProperty $k).PSObject.Properties |
  Where-Object { $_.Name -like 'MicrosoftEdgeAutoLaunch_*' } |
  ForEach-Object { Remove-ItemProperty $k -Name $_.Name }
```

#### 3.2 卸载 DevTools 残包【普通权限即可】

```powershell
Get-AppxPackage -Name 'Microsoft.MicrosoftEdgeDevToolsClient' | Remove-AppxPackage -EA 0
```

#### 3.3 找出旧 WebView2 运行时目录【普通权限列出；删除需管理员】

先列出旧目录，确认它不等于当前登记版本：

```powershell
$pv=(Get-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}' -EA 0).pv
"当前登记版本: $pv"
Get-ChildItem "${env:ProgramFiles(x86)}\Microsoft\EdgeWebView\Application" -Directory |
  Where-Object { $_.Name -match '^\d+\.' -and $_.Name -ne $pv }
```

这些旧目录通常被正在运行的进程占用，**当场删不掉**。重启后，以管理员终端执行下面命令，并把 `$oldPath` 替换为上面列出的旧目录完整路径：

```powershell
$oldPath='<旧目录完整路径>'
if ($oldPath -notlike "${env:ProgramFiles(x86)}\Microsoft\EdgeWebView\Application\*") {
  throw '路径不在预期目录，停止'
}
Remove-Item $oldPath -Recurse -Force
```

> 删除前务必确认：路径位于 `C:\Program Files (x86)\Microsoft\EdgeWebView\Application\` 下，且目录名不等于当前登记版本 `$pv`。误删当前版本会破坏 WebView2。

### 第 4 步 · 收尾

```powershell
Remove-Item "$env:TEMP\edge-uninstall-tool","$env:TEMP\webview2-repair" -Recurse -Force -EA 0
```

可选：`%LOCALAPPDATA%\Microsoft\Edge` 是 Edge 的收藏夹/密码/历史（通常约数百 MB），不会自动删除，按需处理。

---

## 4. 验收清单

```powershell
# Edge 是否彻底没了
Test-Path "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe"   # False
Test-Path "${env:ProgramFiles(x86)}\Microsoft\Edge"                          # False
(Get-Process msedge -EA 0).Count                                             # 0
(Get-AppxPackage | Where-Object { $_.Name -match 'Edge' }).Count             # 0

# WebView2 是否健康
(Get-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}' -EA 0).pv
Get-Service edgeupdate,edgeupdatem -EA 0 | Select-Object Name,Status
Get-AppxPackage Microsoft.Win32WebViewHost | Select-Object Name,Version

# 默认浏览器
(Get-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\Shell\Associations\UrlAssociations\http\UserChoice').ProgId

# Windows Update 仍可扫描（返回数字即正常）
(New-Object -ComObject Microsoft.Update.Session).CreateUpdateSearcher().Search('IsInstalled=0 and IsHidden=0').Updates.Count
```

最后打开一个依赖 WebView2 的程序（电脑管家 / 照片编辑 / Xbox / 任意嵌入式浏览器客户端）确认正常。

**注意**：修复 WebView2 后 `C:\Program Files (x86)\Microsoft\EdgeCore\<版本>\msedge.exe` 会重新出现——那是 WebView2 的 Chromium 引擎，**不是 Edge 回来了**。

---

## 5. 常见坑

| 现象 | 原因 / 处理 |
|---|---|
| github.com 下载超时 | 走 `api.github.com` 资源接口（本手册已内置） |
| 下载的文件凭空消失 | Defender 判毒（Yomal 误报）。先 `Add-MpPreference -ExclusionPath`，完事撤销 |
| WebView2 程序报“运行时未安装” | 卸载工具删了登记，执行第 2 步 |
| 旧运行时目录删不掉 | 被运行中进程占用，重启后删 |
| Windows Update 失败循环 | 原文方案：装回 Edge → 更新完 → 再卸 |
| `DISM /CheckHealth` 报“可修复组件存储” | 与本操作无关，可跑 `DISM /Online /Cleanup-Image /RestoreHealth` |

---

## 6. 回滚

- **装回 Edge**：`winget install Microsoft.Edge` 或微软官网离线安装包
- **WebView2 异常**：重跑第 2 步 Evergreen 安装包

---

## 7. 示例环境参考值（仅作对照，实际环境会不同）

| 项目 | 值 |
|---|---|
| 系统 | Windows 11 25H2（示例） |
| Edge | 158.x（Win32，Program Files (x86)） |
| WebView2 原版本 | 157.x |
| WebView2 修复后 | 158.x |
| Remove-Edge.exe | 2.3，SHA256 `95bdaffa989f146eb0956851b4fbe70540ef6d279bdc9b809950691422ee53d2`（以实际 release 为准） |
| 卸载后释放 | Edge 本体约 1.2 GB + EdgeCore 约 2.7 GB + 旧 WebView2 约 0.9 GB |
| Edge 用户数据 | `%LOCALAPPDATA%\Microsoft\Edge` 约数百 MB，未删 |