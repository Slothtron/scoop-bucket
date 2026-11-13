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
| `qwenwork` | 阿里千问办公（qwenwork.cn）。下载直链是常青地址、文件名不带版本，版本号取自官方更新日志 |
| `winc` | DEEIX 的 Rust 文本编辑器（winc.si，Apache-2.0）。装的是官方 portable zip，`winc` 兼作 `git core.editor` |
| `envboard` | 多环境代理管理器（本仓项目，Tauri 桌面 + 进程内代理引擎，github.com/Slothtron/envboard）。装的是官方免安装 exe；需要 WebView2，状态在 `%LOCALAPPDATA%\envboard` |

`workbuddy-cn` 与 Extras 里的 `workbuddy`（国际版）可以同时安装：两者主程序名不同（`WorkBuddy.exe` / `WorkBuddyAI.exe`），用户数据分别落在 `%APPDATA%\WorkBuddy` 与 `%APPDATA%\WorkBuddy AI`，互不干扰。

`qoder-cn` 装的是解包副本，与官网安装程序共存时共享同一份用户数据（`%APPDATA%\com.qodercn.app.stable`），两者不会互相覆盖，但只应保留一个在用。

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

官网只给一个常青直链 `https://assets.qwenwork.cn/release/latest/qwenworkcn-setup-x64.exe`（electron-builder NSIS 3.04、asInvoker；载荷 `$PLUGINSDIR\app-64.7z` 393 MB，解包 1.5 GB；主程序 `QwenWorkCN.exe`，ProductName `QwenWorkCN`、CompanyName DingTalk）。文件名不带版本，OSS 对象也没有 `x-oss-meta-version` 头，`latest.yml` / `version.json` 一律 404，**版本号只能从官方更新日志取**：`https://docs.qwenwork.cn/release-notes/desktop.md` 里最新一条是 `<Update ... description="千问办公 X.Y.Z">`，清单 `checkver` 就抓这一行（按出现顺序，最新在最前）。`autoupdate` 的 URL 不含 `$version`，哈希由 Excavator 现场下载 393 MB 重算。

安装形态：Scoop 解包 `app-64.7z` 后直接用 `QwenWorkCN.exe`。用户数据在 `%APPDATA%\QwenWorkCN`（另有缓存目录 `~/.qwenworkcn`），与官网安装版共用同一份，只应保留一个在用。应用靠 `resources\install-type.json` 判断安装类型（只认 `user` / `system` / `store` 三个值），scoop 路径命中路径兜底，日志里是 `[InstallType] Resolved Windows install type { installType: 'portable' }`；它没有 Qoder 那种 launcher，也就不需要 `state.ini` / `install.ini` / `updater.cfg`（缺 `updater.cfg` 时 `installerKind` 为 `null`，不影响启动）。

**它会自己建开始菜单入口**：`Programs\千问办公\千问办公.lnk`，target 取 `process.execPath`（版本目录，不是 `current`，升级后就成死链）。压制方式和 qoder-cn 一样——`installer.script` 在该路径放隐藏目录占位、顺手把父目录也设成隐藏，`uninstaller` 删掉；代价是每次启动日志多一条 `Failed to read shortcut link` WARN。菜单里唯一的入口是 `Scoop Apps\千问办公.lnk`。

**自动更新**：`AutoUpdater` / `NativeUpdater` 打 `gateway.qwenwork.cn/api/client/biz/v1/client-updates/check`。未登录时直接 `[NativeUpdater] Skipping check: user not authenticated`；登录后在 `portable` 安装下 `canSilentUpdate=false`，流程是「下载 → 提示 → 用户确认 → 拉起 NSIS」，确认后会往 `%LOCALAPPDATA%\Programs` 再装一份官方 user 布局。**版本只用 `scoop update qwenwork` 升**，别点应用内的更新提示。

`zcode` 与 Extras 同名，注册后 `scoop install zcode` 会变成歧义，要用 `scoop install slothtron/zcode` 指定。`zcode` 与 `zcode-preview` 是同一个应用的两种通道，共用一个 profile（`%APPDATA%\ZCode`），不要同时装两个。

`trae-cn` 与国际版是两条独立发布线（国内版当前号段落后于国际版）。两者的主程序名（`Trae CN.exe` / `Trae.exe`）与 CLI 名（`bin\trae-cn.cmd` / `bin\trae.cmd`）都不同，装完命令是 `trae-cn`，快捷方式也不会撞名。

## 目录结构

```
bucket/      清单（一个应用一个 json）
bin/         维护脚本，转调 scoop 自身的 checkver/checkhashes/checkurls
.github/     Excavator 定时任务
```

## 维护

新版本靠清单里的 `checkver` + `autoupdate` 自动发现：`.github/workflows/excavator.yml` 每 4 小时跑一次，有新版本就按 `autoupdate` 的模板重写 URL、从更新接口取 `sha256hash`，然后提交回仓库。

本地手动跑（在仓库根目录）：

```powershell
.\bin\checkver.ps1 workbuddy-cn        # 只探测
.\bin\checkver.ps1 workbuddy-cn -u     # 探测并重写清单
.\bin\checkhashes.ps1                  # 校验清单哈希与远端文件是否一致
.\bin\checkurls.ps1                    # 校验下载链接是否可达
```

改完清单后 `scoop update` 让本地 bucket 克隆同步，再 `scoop update workbuddy-cn` 升级。
