# 第三方 scoop bucket 里 qoder / trae 的封装调研

2026-11-15 用 GitHub code search（`gh api search/code`）+ raw 直取，抓了 31 份 manifest 做对比。目的：看有没有值得抄回本 bucket 的做法。

## 结论速览

- **Qoder CN 主程序（`Qoder-CN-win-x64-user.exe`）没有任何第三方 scoop bucket 收录**，本 bucket 是独一份。GitHub 上带 `qoder` 的 bucket 全是另外两个产品：`Qoder.json` 是**国际版**（`download.qoder.com/release/latest/QoderUserSetup-x64.exe`），`QoderworkCN.json` 是 **QoderWork CN**（`static.qoder.com.cn/qoder-work-cn/...`，另一个产品）。
- **Trae 国内版有一大堆 bucket**，但高度同质化：31 份里 23 份是 Trae 国内版，按内容哈希只分成 5 个流派，且大多是互相复制（同一份 manifest 出现在 5 个仓库）。
- **值得借鉴的只有两项**，都跟"免下载取版本/哈希"有关，详见下节。其余（261 行 portable profile、innounp 手工解包、persist 重定向）都不适合本 bucket。

## 值得借鉴

### 1. `latest.yml` 能一次给出 Qoder CN 的版本 + sha256 + 版本化 URL（最高价值）

`https://qoder-app.oss-cn-beijing.aliyuncs.com/qoder-app/releases/latest.yml`（`static.qoder.com.cn/qoder-app/releases/latest.yml` 同内容）是 electron-builder 的标准更新清单，实测内容：

```yaml
version: 0.5.11
files:
  - url: https://static.qoder.com.cn/qoder-app/releases/0.5.11/Qoder-CN-win-x64-user.exe
    sha512: ...
    sha256: 88ffcd652573d97c4bfde30eb6dc7a30ae975a58b6d167f8cdfdfb98e5dcc2e4
    size: 268829712
path: Qoder-CN-win-x64-user.exe
releaseDate: 2026-11-13T12:14:10.081Z
```

`sha256` 与 OSS 对象的 `x-oss-meta-sha256`、本清单的 `hash` 三者实测一致。而且 `https://static.qoder.com.cn/qoder-app/releases/0.5.11/Qoder-CN-win-x64-user.exe` **版本化直链存在**（HEAD 206）。

这条线索来自 `nasymonk/homebrew-refill` 的 `Casks/qoder-cn.rb`（它用 `latest-mac.yml` 的 `yaml["version"]` 做版本探测），不是 scoop bucket，但可直接用到我们这里。收益：

- `checkver` 可以从 `script` 型（HEAD 读 `x-oss-meta-version`）换成普通 `url + regex`：`version: ([\d.]+)`；
- `autoupdate` 的 URL 可以带 `$version` 指向版本化副本（不再依赖常青 `latest` 对象在 bump 瞬间被覆盖）；
- `autoupdate.hash` 可以用 `{ "url": <latest.yml>, "regex": "sha256:\\s?([a-f0-9]{64})" }` 或 `jsonpath`（yml 不是 json，得用 regex），让 Excavator **不必每次下载 268 MB 重算哈希**。

这正是本 bucket 历史记录里「autoupdate 不支持 script 型 hash，每次 bump 要重新下载 239 MB」那个包袱的解法。

### 2. checkver 正则的命名捕获组会自动变成 autoupdate 的 `$matchXxx`

Scoop 源码 `lib/autoupdate.ps1:465-469` 把 checkver 正则的命名组（如 `(?<build>...)`）注入替换表，生成 `$matchBuild`：

```powershell
$versionVariables.Add('$match' + (Get-Culture).TextInfo.ToTitleCase($_.Name), $_.Value)
```

`niheaven/scoop-nih` 和 `xrgzs/sdoog` 都用了这个机制：

```json
"checkver":  { "script": [...], "regex": "(?<version>[^=]+)=====(?<url>.+)" },
"autoupdate": { "architecture": { "64bit": { "url": "$matchUrl" } } }
```

——版本和下载地址可以**一次性从接口里成对取出**，autoupdate 直接引用 `$matchUrl`，不用自己拼 URL 模板。Trae 的更新接口 `api.trae.com.cn/icube/api/v1/native/version/trae/cn/latest` 一次返回 `download[].x64`（含 build 号的 URL）和 `versions[].version`（营销版本 3.4.18），`xrgzs/sdoog` 就是用一个 script 同时吐出 `2.3.102387=====<url>` 再拆成两组。

我们当前 `trae-cn.json` 的 `checkver` 走 `jsonpath: $.data.manifest.win32.versions[?(@.region == 'cn')].url` + `regex: stable/(\d+(?:\.\d+)+)/win`，再在 autoupdate 里拼 URL 模板。能工作，但换成 `$matchUrl` 更抗厂商改 URL 结构。

## 不值得借鉴（及原因）

| 做法 | 出处 | 为何不抄 |
|---|---|---|
| **261 行 `post_install` portable profile** | `LaelLuo/scoop`、`kkzzhizhou/scoop-apps`、`cmontage/scoopbucket-third`（三份字节级相同） | 把 `%APPDATA%\Trae CN`、`%LOCALAPPDATA%\Programs\trae-cn`、`~/.trae-cn` 三处搬到 `persist_dir` 再建 junction，还要改 `extensions.json` 里的版本路径。复杂、强依赖厂商目录布局、升级易碎；本 bucket 的 Trae 与官网安装版共存、用户数据留在原位，实测启动正常。 |
| `innosetup` + `extract_dir: {code_GetDestDir}` | 几乎所有 Trae bucket，含本 bucket | 这本来就是我们已在用的，不是新东西。 |
| `pre_install` 手工 `innounpack.exe` | `lzwme/scoop-proxy-cn` 等 2.3.25937 流派 | 旧版产物（`Trae%20CN-Setup-x64.exe`）需要，新版 Inno Setup 用 `innosetup: true` 即可，且它们自己的 `checkver.script` 依赖仓库内 `bin\utils.ps1`（`get_installer_info`），**脱离那个仓库就跑不了**。 |
| 快捷方式带 `--user-data-dir=... --extensions-dir=...` + `persist: data` | lzwme / akirco / kkz | 强制便携模式，会丢掉与官网安装版共享的用户数据；与本 bucket「只保留一个在用」的取向冲突。 |
| winget-pkgs YAML 当哈希源 | lzwme / akirco / SDustDreams | 实测 `manifests/b/ByteDance/Trae/CN/3.4.18/...installer.yaml` 里的 `InstallerSha256` 与我们清单的哈希一致，思路成立；但它们套了 `gh-proxy.org` / `gitproxy.click` 第三方代理，且 winget 用**营销版本号 3.4.18** 建目录、我们清单用 **build 号 2.3.102387**，对不上号（按 build 号拼的路径是 404）。要抄得先解决版本号映射。 |
| `QoderWork CN` 的 `Range: bytes=0-1048575` 读 PE 版本 | kkzzhizhou / cmontage / maboloshi | 只下 1 MB 读 `VersionInfo.FileVersion` 取版本号，很巧，但那是另一个产品（QoderWork），对 Qoder 主程序有现成的 `latest.yml` 可用，不必走这条路。 |
| `bin` shim 指向 `bin\trae-cn.cmd` | 多数 Trae bucket | 我们已经有了。Extras 的 `trae` 用 `bin\trae.cmd`，恰好印证本 bucket 用 `trae-cn.cmd` 避免撞名是对的。 |

## 版本号两套口径（Trae）

Trae CN 更新接口同时给两个数，第三方 bucket 因此分成两派，**都是对的，别当成落后**：

| 口径 | 值 | 用途 | 谁在用 |
|---|---|---|---|
| build 号 | `2.3.102387` | 安装包 URL 路径、Scoop `version` | 本 bucket、LaelLuo/kkz/cmontage/xrgzs、ScoopInstaller Extras |
| 营销版本 | `3.4.18` | 界面显示、winget `PackageVersion`、应用内更新检查 | `niheaven/scoop-nih`（其 `version: 3.4.18` 但 URL 里是 `2.3.102387`）、winget-pkgs |

`niheaven` 的写法是用一个 regex 同时抓两个（`(?<build>...)` + `version"..."([\d.]+)`），autoupdate 只引用 `$matchBuild` 拼 URL——这正好是上面第 2 点的示范。

## 候选清单（按产品归类，均为 `bucket/` 或 `deprecated/` 下的 scoop manifest）

**Qoder CN 主程序**：无（仅本 bucket `qoder-cn.json`）。非 scoop 的相关定义：`nasymonk/homebrew-refill` Cask、`Sittymin/NUR`、`zxp19821005/My_AUR_Files`、`edison7009/EchoBird` 的产品元数据（含 `productId: qoder-cn` / `appId: com.qodercn.app` / `dataDirectoryName: com.qodercn.app.stable` 校验口径）。

**Qoder 国际版**（`QoderUserSetup-x64.exe`，非本产品线）：`lzwme/scoop-proxy-cn`、`kkzzhizhou/scoop-apps`、`akirco/aki-apps`、`anderlli0053/DEV-tools`（后一份是 0.2.29 老版本）。

**QoderWork CN**（另一产品）：`kkzzhizhou/scoop-apps`、`cmontage/scoopbucket-third`、`maboloshi/scoop-private`（三份相同，0.9.18）。

**Trae 国内版**（主要仓库）：`chawyehsu/dorado`、`ScoopInstaller/Extras`（国际版 `trae`）、`lzwme/scoop-proxy-cn`、`kkzzhizhou/scoop-apps`、`arch3rPro/PST-Bucket`、`akirco/aki-apps`、`xrgzs/sdoog`、`cmontage/scoopbucket-third`、`LaelLuo/scoop`、`anderlli0053/DEV-tools`、`niheaven/scoop-nih`、`SDustDreams/scoop-proxy-cn`、`Virace-CDN/PScoop`、`GeorgeDong32/scoopgd`、`FormatToday/scoop-cn`、`Ryanjiena/Breathe`、`dodorz/scoop`、`jqk/scoopbucket`（已 deprecated）。

活跃度参考（stars / 最后 push）：dorado 1239 / 11-15，Extras 2175 / 11-15，kkzzhizhou 547 / 11-14，lzwme 429 / 11-14，PST-Bucket 248 / 11-14——这些仓库都在持续更新，但多数是 Excavator 类机器人同步，不代表人工维护质量。
