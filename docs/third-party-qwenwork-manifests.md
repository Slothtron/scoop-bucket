# 第三方 scoop bucket 里的千问办公（qwenwork）manifest 调研

2026-11-20 用 GitHub code search（`gh api search/code` 搜 `qwenwork` / `qwen-work` / `qwenworkcn` / `qianwenwork`）+ `gh api repos/<repo>/contents/<path> --jq .content | base64 -d`（本环境 raw.githubusercontent.com 被墙）逐份取回。目的：重做本 bucket 的 `qwenwork.json` 前先看现成方案。

## 结论速览

- 共 **11 份 manifest / 8 个仓库**，全部 **x64 only**，都装国内版（`QwenWorkCN.exe`）。分两个流派。
- **流派 A（9 份，字节级相同）**：常青直链 `.../release/latest/qwenworkcn-setup-x64.exe#/dl.7z` + 硬编码 SHA256 + 从官网更新日志取版本。**致命缺陷**：hash 会随常青对象重签而失效，但版本（语义号）没变 → checkver 不触发 → 安装/升级直接失败。本 bucket 的旧清单就是这流派，且**实测已失效**（常青对象 SHA256 从 `f696664a…` 变 `55f98707…`，版本仍 2.1.0）。
- **流派 B（niheaven/scoop-nih，1 份，最稳）**：走厂商升级 API，用精确构建版本 + 固定 CI 产物 + `$matchUrl`。但它的 `autoupdate` 没配 hash extraction，每次 bump 仍下载 398 MB 算 hash。
- **本 bucket 最终方案**：以流派 B 为基础，**补上免下载 hash**——从产物同目录的 `manifest.json` 用 jsonpath 取 MD5。version/URL/hash 全部自动更新，bump 只读几百字节。

## 两个流派

### 流派 A：常青直链 + 7z 解包（9 份，相同）

仓库/路径：
- `lzwme/scoop-proxy-cn` → `bucket/qwenwork-cn.json`、`bucket/QwenWorkCN.json`
- `kkzzhizhou/scoop-apps` → 两个名字都有
- `Scoopforge/Extras-CN` → `bucket/qwenwork-cn.json`
- `SDustDreams/scoop-proxy-cn` → 两个名字都有
- `cmontage/scoopbucket-third` → 两个名字都有
- `dodorz/scoop` → `bucket/QwenWorkCN.json`

字段：version `2.1.0`；URL `https://assets.qwenwork.cn/release/latest/qwenworkcn-setup-x64.exe#/dl.7z`；hash 硬编码 SHA256 `f696664a…`；installer 用 `Expand-7ZipArchive "$dir\`$PLUGINSDIR\app-64.7z"` 解包再清 NSIS 残留；`bin` + `shortcuts` 指向 `QwenWorkCN.exe`；checkver 抓 `docs.qwenwork.cn/release-notes/desktop` HTML 里的版本号；autoupdate URL 同常青直链（无 `$version`）。

问题：autoupdate 只有 URL、没有 hash 更新机制，硬编码 hash 每次发版都会失效，靠人工 bump。且常青对象重签时版本没变，连人工 bump 的触发条件都没有。

### 流派 B：niheaven/scoop-nih（最稳）

`bucket/exclusive/qwenwork-cn.json`：
- version `2.1.0-26111906`（完整构建号，commit 历史显示每日 bump）；
- URL 固定 CI 产物 `https://assets.qwenwork.cn/cicd/qwenworkcn/builds/261119/rel/3bf23777ee6747fb3d50591803e3588e/qwenworkcn_2.1.0-26111906_x64-update-package.7z`；
- hash `fe578d87…`（SHA256）；`extract_dir: app`；
- checkver.script 调 `POST clientupgrade.qwenwork.cn/upgrade/query`，输出 `version url sha512`，regex 三组捕获；
- autoupdate.url = `$matchUrl`。

问题：autoupdate 没配 hash extraction → 每次 bump 下载 398 MB 算 hash（实测 `checkver -u` 打印 `Downloading ... to compute hashes!`）；捕获的 `hash`（sha512 base64）没被用上（Scoop hash 不读 checkver 命名组）。

## 厂商升级 API（本次实测）

```
POST https://clientupgrade.qwenwork.cn/upgrade/query
Content-Type: application/json
{
  "appInfo": {"name":"qwenworkcn","version":"1.0.0","arch":10,"channel":"stable"},
  "osInfo": {"type":0},
  "hardwareInfo": {"machineId":"<GUID>"},
  "userInfo": {"uuid":"<GUID>"},
  "checkType": 0
}
```

响应关键字段（实测）：
```
resultMessage: UPGRADE_HIT
upgradeInfo.version: 2.1.0-26111906
upgradeInfo.resources[0].url: https://assets.qwenwork.cn/cicd/qwenworkcn/builds/261119/rel/3bf23777ee6747fb3d50591803e3588e/qwenworkcn_2.1.0-26111906_x64-update-package.7z
upgradeInfo.resources[0].sha512: 5GaXOTZIodwm/...（base64；解码 hex = e4669739…，文件实测 SHA512 一致）
```

- `resources` 是**数组**，PowerShell `$r.upgradeInfo.resources[0]` 或 `.url` 成员枚举都能取到。
- API 是 POST，Scoop 的 hash extraction 只支持 GET（`WebClient.DownloadData`），所以不能直接用 API 的 sha512 base64 做 autoupdate.hash。
- **关键发现**：CI 产物同目录有可 GET 的 `manifest.json`（把 URL 末段文件名换成 `manifest.json`）：
  ```json
  {
    "releaseVersion": "2.1.0-26111906",
    "artifacts": [
      {"type":"7z","fileName":"qwenworkcn_..._x64-update-package.7z",
       "cdnUrl":"https://assets.qwenwork.cn/cicd/.../qwenworkcn_..._x64-update-package.7z",
       "sizeBytes":417687172,"md5":"b3c5563e612ccc0f1669361381786eee",
       "sha512":"5GaXOTZIodwm/..."}
    ]
  }
  ```
  `artifacts[0].md5` 是可 GET 的 hex MD5，实测与下载文件 `md5sum` 一致。这就是免下载 hash 的来源。

## 本 bucket 最终方案

`bucket/qwenwork.json`（version `2.1.0-26111906`）：

```json
"checkver": {
  "script": [
    "... POST clientupgrade.qwenwork.cn/upgrade/query ...",
    "$url = $r.upgradeInfo.resources[0].url",
    "$hashUrl = ($url -replace '[^/]+$') + 'manifest.json'",
    "Write-Output $r.upgradeInfo.version $url $hashUrl"
  ],
  "regex": "^(\\S+)\\s+(?<url>\\S+)\\s+(?<hashurl>\\S+)$"
},
"autoupdate": {
  "architecture": { "64bit": {
    "url": "$matchUrl",
    "hash": { "url": "$matchHashurl", "jsonpath": "$.artifacts[0].md5" }
  }}
}
```

- 命名组 `url` → `$matchUrl`、`hashurl` → `$matchHashurl`（Scoop `ToTitleCase`）；checkver 无编号组时用 `version` 组当版本（`checkver.ps1:400-406`）。
- `extract_dir: app` 把 7z 内 `app\` 上移到安装根，`QwenWorkCN.exe` 落在 `<dir>`，shortcut 指向它。
- **不再需要**旧版的菜单占位压制：实测删掉 `Programs\千问办公\千问办公.lnk` 后启动新版，应用没有重建它。
- **不再需要** `bin` 字段（第三方流派 A 有，本 bucket 旧版没有；不影响）。
- 官网有 `qwenworkcn-setup-arm64.exe`，但升级 API 与 CI 产物目前只发 x64（arch=10），暂只收 x64。

验证：JSON 全绿；`checkver: 2.1.0-26111906` 幂等；副本模拟旧版后 `checkver -u` 打印 `Found: md5:b3c5563e… using Json Mode`（**没下 398 MB**），version/url/hash/extract_dir 全部正确重建；真实 `scoop update qwenwork` 下载 CI 7z、hash OK、解包、建 `current`、建快捷方式成功；启动得 10 进程 + 窗口「千问办公」+ 不自建菜单项。

## 候选仓库原始路径（本环境 `gh api ... --jq .content | base64 -d`）

- https://raw.githubusercontent.com/niheaven/scoop-nih/HEAD/bucket/exclusive/qwenwork-cn.json （流派 B 底稿）
- https://raw.githubusercontent.com/lzwme/scoop-proxy-cn/HEAD/bucket/qwenwork-cn.json （流派 A）
- https://raw.githubusercontent.com/kkzzhizhou/scoop-apps/HEAD/bucket/qwenwork-cn.json
- https://raw.githubusercontent.com/Scoopforge/Extras-CN/HEAD/bucket/qwenwork-cn.json
- https://raw.githubusercontent.com/SDustDreams/scoop-proxy-cn/HEAD/bucket/qwenwork-cn.json
- https://raw.githubusercontent.com/cmontage/scoopbucket-third/HEAD/bucket/qwenwork-cn.json
- https://raw.githubusercontent.com/dodorz/scoop/HEAD/bucket/QwenWorkCN.json
- 升级 API：POST https://clientupgrade.qwenwork.cn/upgrade/query
- 哈希 sidecar：`<产物同目录>/manifest.json` → `$.artifacts[0].md5`
