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

`workbuddy-cn` 与 Extras 里的 `workbuddy`（国际版）可以同时安装：两者主程序名不同（`WorkBuddy.exe` / `WorkBuddyAI.exe`），用户数据分别落在 `%APPDATA%\WorkBuddy` 与 `%APPDATA%\WorkBuddy AI`，互不干扰。

`qoder-cn` 装的是解包副本，与官网安装程序共存时共享同一份用户数据（`%APPDATA%\com.qodercn.app.stable`），两者不会互相覆盖，但只应保留一个在用。

`zcode` 与 Extras 同名，注册后 `scoop install zcode` 会变成歧义，要用 `scoop install slothtron/zcode` 指定。`zcode` 与 `zcode-preview` 是同一个应用的两种通道，共用一个 profile（`%APPDATA%\ZCode`），不要同时装两个。

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
