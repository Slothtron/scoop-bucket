# scoop-bucket

个人 [Scoop](https://scoop.sh) bucket。

## 安装

```powershell
scoop bucket add slothtron https://github.com/Slothtron/scoop-bucket
scoop install slothtron/workbuddy-cn
```

## 清单

| 清单 | 说明 |
|---|---|
| `workbuddy-cn` | 腾讯 WorkBuddy 国内版（workbuddy.cn）。国际版在官方 Extras bucket，用 `scoop install extras/workbuddy` |
| `qoder-cn` | 阿里 Qoder 国内版（qoder.cn）。官网另有 Qoder CN IDE、QoderWork、QoderWake，未收录 |
| `zcode` | ZCode 稳定通道。与 Extras 的 `zcode` 同一个包，但 checkver 改打厂商更新接口，不再依赖官网页面 |
| `zcode-preview` | ZCode preview 通道，比稳定通道早若干版本 |
| `trae-cn` | 字节 Trae 国内版（trae.cn）。Extras 的 `trae` 是国际版，且其 autoupdate 模板仍用旧产物名 `TraeCode-Setup-x64.exe`，厂商在 2.3.93439 起已改名 `TRAE-Setup-x64.exe` |
| `qwenwork` | 阿里千问办公（qwenwork.cn）。走厂商升级 API 取精确构建版本 + 固定 CI 产物 + 哈希（免下载），Win x64 |
| `feishu` | 字节飞书（feishu.cn）。x64 + 32bit 双架构；版本/下载地址/MD5 全部取自官方 `api/downloads` 接口 |
| `anode` | skulitom/Anode（MIT）。Windows 后台 computer-use MCP server/CLI（独立子会话，给 Claude Code/Codex 用）。x64；portable zip，上游仓库自己也是 scoop bucket |
| `winc` | DEEIX 的 Rust 文本编辑器（winc.si，Apache-2.0）。装的是官方 portable zip，`winc` 兼作 `git core.editor` |
| `envboard` | 多环境代理管理器（本仓项目，Tauri 桌面 + 进程内代理引擎，github.com/Slothtron/envboard）。装的是官方免安装 exe，按**便携形态**运行：`post_install` 写 `envboard.portable` 标记，状态经 `persist` 落到 `scoop\persist\envboard\envboard-data`。需要 WebView2 |

`workbuddy-cn` 与 Extras 里的 `workbuddy`（国际版）可以同时安装：两者主程序名不同（`WorkBuddy.exe` / `WorkBuddyAI.exe`），用户数据分别落在 `%APPDATA%\WorkBuddy` 与 `%APPDATA%\WorkBuddy AI`，互不干扰。

`qoder-cn` 装的是解包副本，与官网安装程序共存时共享同一份用户数据（`%APPDATA%\com.qodercn.app.stable`），两者不会互相覆盖，但只应保留一个在用。

### envboard 的数据目录

`envboard` 按**便携形态**装：清单用 `post_install` 在安装目录写 `envboard.portable` 标记，并用 `persist` 把 `envboard-data` 链到 `scoop\persist\envboard\envboard-data`，升级不丢数据。

- `persist` 不能省：`scoop\apps\envboard\<version>` 在升级/`scoop cleanup` 时会被替换，数据留在版本目录就等于丢。
- 之前按旧清单装过的，数据在 `%LOCALAPPDATA%\envboard`：关掉应用，把该目录内容移到 `scoop\persist\envboard\envboard-data` 再启动，否则环境列表是空的。
- 清单改动不会自动重装：`scoop install envboard --force` 才会重新执行 `persist` 与 `post_install`。
- 0.2.0 起完整支持便携形态：标题带「（portable）」、WebView2 profile 与默认端口段（17000-17999）独立，可与 NSIS 安装的正式版同时运行。0.1.0 只支持「状态随目录走」，同时跑会撞端口。0.2.1 为 UI/UX 整改版本（方向A 设备控制台），形态与数据目录不变。

### qoder-cn 的 launcher 状态文件

官方安装程序会写 `%LOCALAPPDATA%\Qoder CN\Qoder CN Launcher\state.ini`，0.5.1 起主程序启动时强制读取它，缺失就报 `WINDOWS_LAUNCHER_STATE_MISSING`（界面上表现为「应用服务无法完成初始化，本地数据没有被自动覆盖」）。Scoop 是直接把 NSIS 包里的 `app*.7z` 解包，安装器不运行，这个文件不会生成。

所以清单的 `installer.script` 在解包后补写该文件，`targetVersion` 用 `$version`。若目标位置已存在 `state.ini`（例如之前装过官方版），首次写入前会另存为 `state.ini.bak`，之后不再覆盖该备份。

同目录还要有 **`install.ini`**（官方安装器写的「稳定层」状态，`installer.script` 同样补写，备份规则同 `state.ini`）。launcher 启动时把两份合并：只有 `install.ini` 里存在 `installDir` 时才会采纳 `state.ini` 的 `appExecutable`，缺 `install.ini` 则合并结果里应用路径为空，点快捷方式弹「未找到已安装的 Qoder CN。重新运行安装包以修复应用。」。两份的 `installDir` 必须一致，否则判 mismatch。

`installDir` **必须写 `current` 链接的路径**，不能写 `$dir`（版本目录）：0.5.5 起主程序把状态文件里的 `installDir` 和自己实际所在目录做字符串比较，走 `current` 启动时看到的是 `...\apps\qoder-cn\current`，与版本目录 `...\apps\qoder-cn\0.5.5` 不相等就报 `WINDOWS_LAUNCHER_INSTALLATION_MISMATCH` 并退出。清单里用的是 `Join-Path (Split-Path $dir -Parent) 'current'`，跨版本恒定。

同理，**启动 qoder-cn 要经由 `current`**（开始菜单快捷方式、`scoop run qoder-cn` 或 shim）。直接点 `apps\qoder-cn\<version>\Qoder CN.exe` 时应用看到的路径是版本目录，会反过来判定 mismatch。

卸载 `qoder-cn` 不会删除这两份状态文件，重装时会被重写；若要恢复官方版留下的记录，把 `state.ini.bak` / `install.ini.bak` 改名回去即可。

### qoder-cn 的内置自动更新不要用

0.5.5 起应用自己带桌面更新，会去 `gateway.qoder.com.cn/algo/api/update/desktop/win32-x64-user/stable/<version>` 查版本，并优先走 fast-update（把 launcher 复制到候选目录再校验）。**这条链路在 scoop 布局下必然失败**：

```
[WARN] Windows fast update staging failed; falling back to NSIS
  {"version":"0.5.6","failureStage":"candidate_prepare",
   "error":"WINDOWS_LAUNCHER_PREPARATION_FAILED:
            refresh=WINDOWS_LAUNCHER_LOCALE_UNAVAILABLE:zh-CN.properties:
            reuse=WINDOWS_LAUNCHER_LOCALE_UNAVAILABLE:zh-CN.properties"}
```

失败点不在文件缺失——`resources/fast-update/qoder-fast-update-manifest.json` 里 launcher 的三个条目（`Qoder CN Launcher.exe`、`i18n/en-US.properties`、`i18n/zh-CN.properties`）哈希与安装目录逐个对得上；单独把 `Qoder CN Launcher.exe` 复制到空目录运行也正常。是厂商的候选目录 staging 逻辑在 `current` 软链布局下拿不到 locale 文件。

失败后会**回落到 NSIS**：下载 271 MB 的 `Qoder-CN-win-x64-user.exe` 到 `%LOCALAPPDATA%\qoder-cn-updater\attempts\<uuid>\pending\`，状态置 `ready`。这个安装包装的是官方 user 布局（`%LOCALAPPDATA%\Programs\...`），会在 scoop 这份之外再装一份，和 `current` 抢启动入口。

**所以：版本只用 `scoop update qoder-cn` 升。** 已经暂存过安装包的机器，删掉暂存目录即可（可重新下载，不影响已装版本）：

```powershell
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\qoder-cn-updater"
```

### 内置自动更新的处置

**清单已把它整体关掉**：`installer.script` 把 `resources\install-type.json` 写成 `{"installType":"portable"}` 后，桌面更新检查直接禁用（日志出现 `Background Desktop update checks are disabled`），不会再产生 assignment、更不会回落 NSIS 下载 271 MB。下面保留原始排查记录——在默认的 `{"installType":"user"}` 形态下，厂商没有提供关闭开关，只能「保持最新」：

- **`%APPDATA%\com.qodercn.app.stable\updates\v1\state.json` 不是控制面**。把 `stage` 改成 `"disabled"` 会在启动时被重置回 `"idle"`，`skippedVersions` 也会被服务端 assignment 覆盖，随后照常下载。
- **应用内「跳过此版本」只对当前版本生效**（服务端 `policy.allowSkip: true`），下一个版本仍会提示。

真正决定「要不要下载」的是服务端 assignment：

| 本机版本 | assignment | 行为 |
|---|---|---|
| 落后（0.5.5 vs 0.5.6） | `server-0.5.6-<revision>` | 下载 271 MB → fast-update 失败 → 回落 NSIS |
| 最新（0.5.6 vs 0.5.6） | `server-hold-0.5.6-<ts>` | 什么都不做 |

**只要 bucket 跟得上厂商发版，自动更新就不会触发**——这正是 `.github/workflows/excavator.yml` 每 4 小时跑一次的意义。判断当前状态看这行日志即可：

```powershell
Select-String -Path "$env:APPDATA\com.qodercn.app.stable\logs\*\main.log" -Pattern 'assignment-check"'
# assignmentId 以 server-hold- 开头 = 无更新；server-<version>- 开头 = 会下载
```

应用启动时会自己重建 `开始菜单\Programs\Qoder CN.lnk`（指向 `%LOCALAPPDATA%\Qoder CN\Qoder CN Launcher\Qoder CN Launcher.exe`，工作目录 `current`）。这不是第二份安装，它读的就是我们写的 `state.ini`，`installDir` 写对了它也能正常启动；不过该行为已被清单压制，见下节。

### 禁止应用自建开始菜单入口

`Programs\Qoder CN.lnk` 由应用启动时的「应用图标投影」环节重建（目标指向 exe、图标写到 `%APPDATA%\com.qodercn.app.stable\application-icons\`），**没有官方开关**——设置里、`state.json` 里都翻不到。反编译 `electronMain.js` 能确认：名为 `cQe` 的快捷方式 ensure 流程只有 `install-type.json` 为 `user`/`system` 时才走创建分支（缺省安装类型下它反而只修不建），但图标投影另有一条重建路径。清单用两层手段压制：

1. **`resources\install-type.json` 写成 `{"installType":"portable"}`**：关掉 `cQe` 创建分支与旧路径迁移，顺带禁用桌面更新检查（见上节）；
2. **隐藏目录占位**：在 `%APPDATA%\Microsoft\Windows\Start Menu\Programs\` 建一个名为 `Qoder CN.lnk` 的**隐藏目录**——同名文件创建必然失败，隐藏目录不出现在开始菜单应用列表，应用侧把错误静默吞掉，启动无报错。

`installer.script` 每次安装/升级都会重建占位，`uninstaller` 卸载时删掉它。菜单里唯一的 Qoder 入口是 `Scoop Apps\Qoder CN.lnk`（scoop 清单 `shortcuts` 字段创建），`scoop reset qoder-cn` 管的就是它。

### 千问办公（qwenwork）

**版本 / 地址 / 哈希全部来自厂商升级 API**：`POST https://clientupgrade.qwenwork.cn/upgrade/query`（body 里 `appInfo.name=qwenworkcn`、`arch=10`、`channel=stable`，machineId/uuid 用随机 GUID）。响应 `upgradeInfo` 一次给出精确构建版本（如 `2.1.0-26111906`）、`resources[0].url`（**固定 CI 产物**，不是常青包）和 `sha512`(base64)。`checkver.script` 调这个 API，输出 `版本 URL 同目录 manifest.json URL` 三元组，用命名组捕获；`autoupdate.url = $matchUrl`，哈希用 `autoupdate.hash = { url: $matchHashurl, jsonpath: $.artifacts[0].md5 }` 从产物同目录的 `manifest.json` 取 MD5。

比旧的「官网常青直链 + 更新日志取版本」方案强在两点：

1. **根治 hash 静默失效**：旧清单用 `release/latest/qwenworkcn-setup-x64.exe` 常青地址 + 硬编码 hash + 从更新日志取语义版本。实测该常青对象在版本没变时就重签过（SHA256 从 `f696664a…` 变 `55f98707…`），导致清单 hash 对不上、`checkver` 又因版本没变不触发——`scoop install/update` 直接失败。新方案版本用精确构建号、地址用不可变 CI 产物，不会再错位。
2. **Excavator bump 免下载**：产物同目录有 `manifest.json`，里面 `artifacts[0].md5` 是可 GET 的 hex MD5（与文件实测一致）。`autoupdate.hash` 用 `jsonpath` 取它，bump 时只读几百字节 JSON，不再每次下 398 MB。实测 `checkver -u` 打印 `Found: md5:b3c5563e… using Json Mode`，版本/URL/hash 全部正确重建。

**安装形态**：CI 产物是纯 7z（`qwenworkcn_..._x64-update-package.7z`，约 399 MB），内部自带 `app\` 前缀；`extract_dir: app` 会把内容上移到安装根，主程序落在 `<dir>\QwenWorkCN.exe`，快捷方式指向它。用户数据仍在 `%APPDATA%\QwenWorkCN`，与官网安装版共用同一份，只应保留一个在用。**新版不再自建开始菜单入口**：实测删掉 `Programs\千问办公\千问办公.lnk` 占位后启动，应用没有重建它——所以清单不再需要旧版的 installer/uninstaller 隐藏目录占位逻辑。菜单里唯一的入口是 `Scoop Apps\千问办公.lnk`。

**自动更新**：内置 updater 打 `gateway.qwenwork.cn/api/client/biz/v1/client-updates/check`；未登录不检查，登录后在 portable 安装下只能手动确认并拉起 NSIS 装到 `%LOCALAPPDATA%\Programs`。**版本只用 `scoop update qwenwork` 升**，别点应用内的更新提示。官网另提供 `qwenworkcn-setup-arm64.exe`，但厂商升级 API 与 CI 产物目前只发 x64（arch=10），故清单暂只收 x64。

### 飞书（feishu）

下载页 `feishu.cn/download` 是 Vite SPA，直链不在 HTML 里；它运行时打 `https://www.feishu.cn/api/downloads`，一次返回全部平台的版本、直链和哈希。这是本项目里元信息最全的一家：

```json
"Windows":    { "download_link": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/2ea298b4/Feishu-win32_ia32-8.6.13-signed.exe", "version_number": "Windows@V8.6.13", "hash": "a54ac30c11b4204484790b032668d725" },
"Windows_x64":{ "download_link": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/66bd5227/Feishu-win32_x64-8.6.13-signed.exe", "version_number": "Windows@V8.6.13", "hash": "f5202819cb9a06cceeb138de42c970d0" }
```

- **版本 / 地址 / 哈希三合一**：`checkver` 用一条正则同时抓版本号和两段 CDN 路径（`path` = ia32、`pathx` = x64），`autoupdate` 用 `$matchPath` / `$matchPathx` + `$version` 重建两个架构的 URL，哈希用 `jsonpath` 从同一接口的 `$.versions.Windows.hash` / `$.versions.Windows_x64.hash` 取——Excavator bump 只读一个 JSON，不用下载 491 MB 安装包。
- **哈希是 MD5**（32 位裸值），清单里写 `md5:` 前缀；实测 `md5sum` 与接口值一致（`f5202819…`）。
- **两个可用的接口**：`/api/package_info?platform=16`（x64）/ `?platform=7`（ia32）返回**带签名的临时 URL**（`x-expires` 约 30 分钟，过期后 403，改 `x-expires` 会破坏签名）；`/api/downloads` 给的是 `sf3-cn.feishucdn.com/obj/...` 的**无签名稳定直链**，清单用后者。
- **两个架构都收**：`Windows`（ia32）和 `Windows_x64`；飞书官方不提供 arm64 桌面版。
- **安装形态**：下载的是 491 MB 自解压 exe（内含 Delta LZMA 的 7z 载荷），用 `#/dl.7z` 让 Scoop 直接解包；`extract_to: app` 把 9896 个文件隔离到 `app\`，主程序 `app\Feishu.exe`，快捷方式指向它。用户数据走应用默认目录，与官网安装版共用同一份，只应保留一个在用。

**自动更新的开关落点与禁用**：设置页「自动安装更新」复选框读写 preference 键 `lark.update.disable_auto_install_update`（关闭时写 `true`），存在 `%APPDATA%\LarkShell\sdk_storage\<用户 hash>\settings.db`——这是**加密的 global preference**（文件头非 SQLite magic，全盘字面搜不到键名），用户级、跨 scoop 升级保留。关闭动作本身被记进 `%APPDATA%\LarkShell\update\update_report.json`，事件名 `update_disabled_install`。

从 `frame.dll` 反解出飞书更新器还支持两个官方禁用信号：环境变量 **`LARK_UPDATE_DISALLOW_AUTO_UPDATE`**（日志 `LARK_UPDATE_DISALLOW_AUTO_UPDATE is set, won't start update`）和 flag 文件 `disallow_auto_update.flag`。清单用 Scoop 原生 `env_set` 字段设 `LARK_UPDATE_DISALLOW_AUTO_UPDATE=1`：用户级环境变量、卸载时 Scoop 自动清理，且实测重启后 `update\`/`update_downloading\` 目录 mtime 不变（不再下载更新），应用照常启动。`env_set` 与 version/URL/hash 无关，Excavator bump 不会动它（已在副本上模拟 bump 验证）。

- 第三方 bucket 里 feishu manifest 很多（dorado / kkzzhizhou / lzwme 等 14 个仓库、30 份），本清单的 checkver/autoupdate 以 `chawyehsu/dorado` 为基础；调研细节见 `docs/third-party-feishu-manifests.md`。

`zcode` 与 Extras 同名，注册后 `scoop install zcode` 会变成歧义，要用 `scoop install slothtron/zcode` 指定。`zcode` 与 `zcode-preview` 是同一个应用的两种通道，共用一个 profile（`%APPDATA%\ZCode`），不要同时装两个。

`trae-cn` 与国际版是两条独立发布线（国内版当前号段落后于国际版）。两者的主程序名（`Trae CN.exe` / `Trae.exe`）与 CLI 名（`bin\trae-cn.cmd` / `bin\trae.cmd`）都不同，装完命令是 `trae-cn`，快捷方式也不会撞名。

### Anode（skulitom/Anode）

Windows 后台 computer-use 工具（MIT，C#/.NET 10）：给 Claude Code / Codex 等 stdio MCP 客户端一个**独立的子会话**（child session / loopback RDP），agent 在里面操作原生应用、截图、做 UI 测试，不抢用户鼠标键盘。**仅 x64、Windows Pro/Enterprise/Education（Home 不支持）**，需一次性管理员 `anode setup` 开启 child sessions（会重启 Remote Desktop Services）。完整源码走读见 `research/anode-2026-11/anode-analysis.md`。

上游仓库**自己就是 scoop bucket**（`scoop bucket add anode https://github.com/skulitom/Anode`），`bucket/anode.json` 维护得很规范；本清单以它为底稿，两个关键点：

- **不用 `bin` shim**：`anode.exe` 是 WinExe（GUI 子系统但保留 stdout），Scoop shim 在启动前释放控制台，CLI 输出会丢。改用 `env_add_path: "."` 把安装目录加进 PATH（安装后需新开终端才生效）。实测 `anode.exe --version` → `anode 0.11.4`。
- `checkver: "github"` + autoupdate hash 从同 release 的 `SHA256SUMS` 用 Extract Mode 提取，Excavator bump 只下一个几百字节文本、不碰 71 MB 的 zip。
- **升级前要 `anode quit`** 并关掉 MCP 客户端，否则 scoop 因 exe 被占用拒绝更新（notes 里已写明）。

## 目录结构

```
bucket/      清单（一个应用一个 json）
bin/         维护脚本，转调 scoop 自身的 checkver/checkhashes/checkurls
.github/     Excavator 定时任务
```

## 维护

新版本靠清单里的 `checkver` + `autoupdate` 自动发现：`.github/workflows/excavator.yml` 每 4 小时跑一次，有新版本就按 `autoupdate` 的模板重写 URL 与哈希，然后提交回仓库。

`qoder-cn` 走的是厂商的 electron-builder 更新清单 `qoder-app.oss-cn-beijing.aliyuncs.com/qoder-app/releases/latest.yml`：它同时给 `version`、`files[0].url`（**版本化直链**）和 `files[0].sha256`，所以 `checkver` 用普通 `url + regex: version: ([\d.]+)`，`autoupdate.architecture.64bit.url` 用 `$version` 拼 `static.qoder.com.cn/qoder-app/releases/$version/...`，`autoupdate.hash` 直接从 yml 里抽 sha256。**Excavator bump 不再需要下载 256 MB 重算哈希**，也不用担心常青 `latest` 对象在发版瞬间被覆盖。哈希正则用 `(?s)files:.*?sha256:…` 锚定 `files[0]`，避开 `differential` / `windowsFastUpdate` 里同名字段。版本化对象与 `latest` 实测 ETag 相同。

`trae-cn` 的更新接口一次返回「build 号」（`2.3.102387`，用于 URL 与 Scoop `version`）和「营销版本」（`3.4.18`，界面/winget 用）两个数，清单取前者；`checkver` 走 `url + jsonpath + regex`，`autoupdate` 用 `$version` 拼 URL。Scoop 也支持把 checkver 正则的命名捕获组（如 `(?<url>…)`）自动注入成 `$matchUrl` 直接引用接口给的下载地址（`lib/autoupdate.ps1:464-469`），但那需要改成 script 型 `checkver`，收益抵不上复杂度，本清单不用——细节见 `docs/third-party-qoder-trae-manifests.md`。

本地手动跑（在仓库根目录）：

```powershell
.\bin\checkver.ps1 workbuddy-cn        # 只探测
.\bin\checkver.ps1 workbuddy-cn -u     # 探测并重写清单
.\bin\checkhashes.ps1                  # 校验清单哈希与远端文件是否一致
.\bin\checkurls.ps1                    # 校验下载链接是否可达
```

改完清单后 `scoop update` 让本地 bucket 克隆同步，再 `scoop update workbuddy-cn` 升级。
