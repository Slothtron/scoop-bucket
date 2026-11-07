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
| `winc` | DEEIX 的 Rust 文本编辑器（winc.si，Apache-2.0）。装的是官方 portable zip，`winc` 兼作 `git core.editor` |

`workbuddy-cn` 与 Extras 里的 `workbuddy`（国际版）可以同时安装：两者主程序名不同（`WorkBuddy.exe` / `WorkBuddyAI.exe`），用户数据分别落在 `%APPDATA%\WorkBuddy` 与 `%APPDATA%\WorkBuddy AI`，互不干扰。

`qoder-cn` 装的是解包副本，与官网安装程序共存时共享同一份用户数据（`%APPDATA%\com.qodercn.app.stable`），两者不会互相覆盖，但只应保留一个在用。

### qoder-cn 的 launcher 状态文件

官方安装程序会写 `%LOCALAPPDATA%\Qoder CN\Qoder CN Launcher\state.ini`，0.5.1 起主程序启动时强制读取它，缺失就报 `WINDOWS_LAUNCHER_STATE_MISSING`（界面上表现为「应用服务无法完成初始化，本地数据没有被自动覆盖」）。Scoop 是直接把 NSIS 包里的 `app*.7z` 解包，安装器不运行，这个文件不会生成。

所以清单的 `installer.script` 在解包后补写该文件，`targetVersion` 用 `$version`。若目标位置已存在 `state.ini`（例如之前装过官方版），首次写入前会另存为 `state.ini.bak`，之后不再覆盖该备份。

`installDir` **必须写 `current` 链接的路径**，不能写 `$dir`（版本目录）：0.5.5 起主程序把状态文件里的 `installDir` 和自己实际所在目录做字符串比较，走 `current` 启动时看到的是 `...\apps\qoder-cn\current`，与版本目录 `...\apps\qoder-cn\0.5.5` 不相等就报 `WINDOWS_LAUNCHER_INSTALLATION_MISMATCH` 并退出。清单里用的是 `Join-Path (Split-Path $dir -Parent) 'current'`，跨版本恒定。

同理，**启动 qoder-cn 要经由 `current`**（开始菜单快捷方式、`scoop run qoder-cn` 或 shim）。直接点 `apps\qoder-cn\<version>\Qoder CN.exe` 时应用看到的路径是版本目录，会反过来判定 mismatch。

卸载 `qoder-cn` 不会删除这个状态文件，重装时会被重写；若要恢复官方版留下的记录，把 `state.ini.bak` 改名回 `state.ini` 即可。

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

### 关不掉，只能「保持最新」

厂商没有提供关闭自动更新的开关，实测两条直觉做法都无效：

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

应用启动时会自己重建 `开始菜单\Programs\Qoder CN.lnk`（指向 `%LOCALAPPDATA%\Qoder CN\Qoder CN Launcher\Qoder CN Launcher.exe`，工作目录 `current`）。这不是第二份安装，删掉也会长回来；它读的就是我们写的 `state.ini`，`installDir` 写对了它也能正常启动。

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
