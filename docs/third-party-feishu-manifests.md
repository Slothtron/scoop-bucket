# 第三方 scoop bucket 里的飞书（feishu）manifest 调研

2026-11-19 用 GitHub code search（`gh api search/code?q=feishu+in:file+extension:json+path:bucket`，74 命中）+ `gh api repos/<repo>/contents/...` 逐份取回，共 30 份 manifest / 14 个仓库。目的：做本 bucket 的 `feishu.json` 前，先看现成方案并挑可借鉴的。

## 结论速览

- **现成方案高度同质**：30 份里绝大多数版本 8.6.13、下载地址 `sf3-cn.feishucdn.com/obj/ee-appcenter/<id>/Feishu-win32_*-signed.exe`、哈希取自飞书官方接口、用 `#/dl.7z` 解包、`extract_to: app`、快捷方式指向 `app/Feishu.exe`。
- **元数据接口有两套**：`/api/package_info?platform=<n>`（带签名临时 URL）和 `/api/downloads`（无签名稳定 URL + 全平台）。现成 manifest 基本都用后者。
- **最稳的是 `chawyehsu/dorado` 的 `bucket/feishu.json`**：双架构、命名组 checkver、`jsonpath` 取哈希、1.2k stars 且当天还有 push。本 bucket 的清单以它为底稿。
- **没有一份支持 arm64**：飞书官方桌面版只有 ia32 / x64 两个 Windows 产物。

## 官方接口与产物（本次实测）

`https://www.feishu.cn/api/downloads` 返回 `versions` **对象**（不是数组），Windows 相关字段：

```json
"Windows":     { "download_link": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/2ea298b4/Feishu-win32_ia32-8.6.13-signed.exe", "version_number": "Windows@V8.6.13", "hash": "a54ac30c11b4204484790b032668d725", "release_time": "1794486023" },
"Windows_x64": { "download_link": "https://sf3-cn.feishucdn.com/obj/ee-appcenter/66bd5227/Feishu-win32_x64-8.6.13-signed.exe", "version_number": "Windows@V8.6.13", "hash": "f5202819cb9a06cceeb138de42c970d0", "release_time": "1794486023" }
```

另一套接口 `https://www.feishu.cn/api/package_info?platform=16`（x64）/ `?platform=7`（ia32）也返回 `download_link` / `version_number` / `hash`，但 `download_link` 是**带签名的临时 URL**：

- `?lk3s=...&x-expires=<unix>&x-signature=...`，`x-expires` 约 30 分钟后到期；实测过期后改大 `x-expires` 或去掉签名都返回 403（Tengine CDN 校验），无签名裸路径 `/ee-appcenter/<id>/...` 也 403。
- 所以**不能把签名 URL 写进 manifest**；要用 `/api/downloads` 里 `sf3-cn.feishucdn.com/obj/...` 的无签名稳定直链。

哈希：`hash` 是 **32 位裸 MD5**，实测下载 x64 安装包 491 MB 后 `md5sum = f5202819cb9a06cceeb138de42c970d0`，与接口一致；清单写 `md5:` 前缀。（注意 ETag 的 md5 段 `105216d2…` 不是文件 MD5，别用 ETag。）

产物形态：491 MB PE32+ exe，内含 Delta LZMA:26 BCJ2 的 7z 载荷（7z 检测为 `Type = 7z`，位于 `.rsrc\2052\ZIP\133`）。用 `#/dl.7z` 直接解包即可，得到 `Feishu.exe` + 9896 个文件（解包约 2.2 GB）；`extract_to: app` 后主程序在 `app\Feishu.exe`。哈希校验由 Scoop 在下载后完成，实测 `Checking hash ... OK`。

## 现成 manifest 分类

**双架构（x64 + ia32，当前 8.6.13）**：`chawyehsu/dorado`、`kkzzhizhou/scoop-apps`、`cmontage/scoopbucket-third`（含 variant `feishu_scoopcn.json`）、`lzwme/scoop-proxy-cn`、`scoopcn/scoopcn`、`SDustDreams/scoop-proxy-cn`、`FormatToday/scoop-cn`、`xrgzs/sdoog`（带数据目录处理）、`anderlli0053/DEV-tools`（7.60.8，旧）。

**仅 x64（8.6.13）**：`Scoopforge/Extras-CN`、`LaelLuo/scoop`（带 portable profile）、`cmontage/feishu_Scoopforge.json`、`cmontage/feishu_LaelLuo.json`、`kkzzhizhou/feishu_Scoopforge.json`、`kkzzhizhou/feishu_LaelLuo.json`、`Jadeiin/scoop`（顶层单 ia32 URL）。

**单架构 / 老版本**：`akirco/aki-apps`、`jqk/scoopbucket`、`okibcn/ScoopMaster`、`anderlli0053/AppArchive`（顶层 ia32 URL + regex 拼 autoupdate）。

**未收录 feishu**：`ScoopInstaller/Extras`、`arch3rPro/PST-Bucket`、`niheaven/scoop-nih`、`maboloshi/scoop-private`、`GeorgeDong32/scoopgd`、`Virace-CDN/PScoop`。

## 值得借鉴

1. **`#/dl.7z` URL 片段**：签名 exe 实为自解压 7z，片段让 Scoop 直接按 7z 解包，省掉 Inno Setup / NSIS 静默参数。几乎所有现成 manifest 都这么做。
2. **`extract_to: "app"` + 快捷方式指向 `app/Feishu.exe`**：把 9896 个文件隔离到子目录，快捷方式路径稳定。
3. **哈希直接从元数据 API 取**：`autoupdate.hash = {url: https://www.feishu.cn/api/downloads, jsonpath: $.versions.Windows_x64.hash}`，Excavator bump 只读 JSON，不重下 491 MB。
4. **命名组同时抓版本 + 两段 CDN 路径**：dorado 的 `(?sm)Windows.*?https://(?<path>...)/Feishu.*?-(?<version>[\d.]+).*?.exe.*?Windows_x64.*?https://(?<pathx>...)/Feishu` 一次给出 `$version` / `$matchPath` / `$matchPathx`；Scoop `checkver` 在无编号组时把 `version` 组当版本（`checkver.ps1:400-406` 实测），命名组注入为 `$match<Group>`。
5. **`jp` 是 `jsonpath` 的短别名**：Scoop 两者都支持。
6. **数据目录处理两种流派**（本清单未采用，Feishu 默认数据目录即可用）：
   - `LaelLuo/scoop` / `cmontage/feishu_LaelLuo.json`：`persist: data` + `post_install` 把 `%APPDATA%\LarkShell`、`%LOCALAPPDATA%\Feishu` 用 `CreateSymbolicLinkW`（带 `SYMBOLIC_LINK_FLAG_ALLOW_UNPRIVILEGED_CREATE 0x2`）链进 persist 目录，且先探测能力再断链，避免开发者模式未开时破坏数据。
   - `xrgzs/sdoog`：`installer.script` 用 `New-PersistDirectory "$env:APPDATA\LarkShell" "$persist_dir\AppData" -Migrate`，更简洁。
7. **winget 作为哈希兜底**（`anderlli0053/DEV-tools/FeiShu_abgox.json`）：`checkver.script` 调仓库 helper `get_installer_info 'ByteDance/Feishu'`，`autoupdate.hash` 从 winget 的 `ByteDance.Feishu.installer.yaml` 正则取 `InstallerSha256`（SHA-256，x86 only）。飞书自有接口够用，未采用。

## 不借鉴

| 做法 | 出处 | 原因 |
|---|---|---|
| `(Invoke-WebRequest ...).versions[0]` | lzwme / SDustDreams / scoopcn / kkzzhizhou 的 feishu_scoopcn | 现接口 `versions` 是对象不是数组，`[0]` 假设脆弱/已坏 |
| 只收 x64 或顶层单 ia32 URL | Scoopforge/Extras-CN、Jadeiin、akirco、jqk | 飞书有双架构，没理由丢一半 |
| 261 行 portable profile | LaelLuo / cmontage/feishu_LaelLuo | 与本 bucket「用户数据留默认位置、与官网版只留一个在用」取向冲突 |
| 带签名的 `download_link` 直接写进 URL | `/api/package_info` 的原始输出 | 30 分钟过期，改参数即 403 |

## 本 bucket 的落地

`bucket/feishu.json` 以 dorado 为底稿，版本 8.6.13：

- 双架构 x64 + ia32；`hash` 用 `md5:` 前缀（接口给的是 MD5）；
- `checkver` 用 dorado 的命名组正则，`autoupdate` 用 `$matchPath`/`$matchPathx` + `$version` 拼无签名稳定 URL，哈希用 `jsonpath` 取同一接口；
- `#/dl.7z` + `extract_to: app` + 快捷方式 `app/Feishu.exe` → 快捷方式名「飞书」。

验证：JSON 全绿；`checkver feishu: 8.6.13` 幂等；把版本改成 8.0.0 + 假 URL/假哈希后 `checkver -u` 正确 bump 回 8.6.13 并重建两个 URL + 两个 MD5（复制品上测）；真实 `scoop install feishu` 下载 491 MB、哈希 OK、解包、建 `current`、建快捷方式成功；启动 `app\Feishu.exe` 得到 8 个进程 + 窗口标题「飞书」、零错误对话框。

### 自动更新开关的落点（2026-11-20 补充）

用户在设置页关掉「自动安装更新」后，排查配置落点：

- **UI 开关 = preference 键 `lark.update.disable_auto_install_update`**。从 `app\webcontent\setting.asar` 的设置页代码反解：`componentDidMount` 读 `T.Jt("lark.update.disable_auto_install_update")`，`toggleAutoInstallUpdate` 写 `T.hZ("lark.update.disable_auto_install_update", !e)`（关闭时写 `true`）。这是 setting.asar 里**唯一**的 `lark.update.*` 键。
- **存储 = 加密的 global preference**。模块 96214 的 `J8()` 分支（`isPreferenceContainerEnabled && isLarkAPISettingsMigrationEnabled`）走 `LarkAPI.preference.getGlobalPreference()`，落在 `%APPDATA%\LarkShell\sdk_storage\<用户 hash>\settings.db`。该文件**加密**（头部 `k\xc5\xe5\x96…` 非 SQLite magic），全盘字面/二进制搜不到键名；`settings.db-wal` 也加密。用户级、跨 scoop 升级保留。
- **另一个相邻键 `lark.update.disable_auto_update`**（frame.dll 里也有），但 UI 复选框用的是 `disable_auto_install_update`。
- **关闭动作留痕**：`%APPDATA%\LarkShell\update\update_report.json` 里 `event_name: update_disabled_install`，时间戳即关闭时刻。此前飞书已据此下载过 8.7.8 全量更新包到 `update\update\8.7.8\`（491 MB 级，未安装）。
- **frame.dll 里的官方禁用信号**：环境变量 `LARK_UPDATE_DISALLOW_AUTO_UPDATE`（`... is set, won't start update`）、flag 文件 `disallow_auto_update.flag`（`Disallow auto update flag file exists ..., won't start update`），另有 `LARK_UPDATE_ALLOW_DEV_AUTO_QUERY` 等。还有 `ShellAPI::app::update::DisableAutoUpdate` / `queryDisableStatus` API。

**清单落地**：用 Scoop 原生 `env_set` 字段设 `LARK_UPDATE_DISALLOW_AUTO_UPDATE=1`（用户级环境变量，卸载自动清理）。实测：设该变量并重启后飞书照常启动（8 进程、窗口「飞书」），`update\`/`update_downloading\` mtime 不变（没再下载更新）。`env_set` 与 version/url/hash 无关，副本上模拟 `checkver -u` bump 后 `env_set` 原样保留。备选方案（未采用）：删除安装目录的 `update.exe`（实测也能启动，但升级会重新解包带回，且不如环境变量干净）、放 `disallow_auto_update.flag`（路径未 100% 钉死）。

## 候选仓库原始路径（本环境用 `gh api ... --jq .content | base64 -d` 取回）

- https://raw.githubusercontent.com/chawyehsu/dorado/HEAD/bucket/feishu.json （底稿）
- https://raw.githubusercontent.com/kkzzhizhou/scoop-apps/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/lzwme/scoop-proxy-cn/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/xrgzs/sdoog/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/LaelLuo/scoop/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/Scoopforge/Extras-CN/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/scoopcn/scoopcn/HEAD/bucket/feishu.json
- https://raw.githubusercontent.com/anderlli0053/DEV-tools/HEAD/bucket/FeiShu_abgox.json
- 元数据接口：https://www.feishu.cn/api/downloads 、https://www.feishu.cn/api/package_info?platform=16
- winget 哈希兜底：https://raw.githubusercontent.com/microsoft/winget-pkgs/master/manifests/b/ByteDance/Feishu/$version/ByteDance.Feishu.installer.yaml
