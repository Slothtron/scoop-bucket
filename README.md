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

`workbuddy-cn` 与 Extras 里的 `workbuddy`（国际版）可以同时安装：两者主程序名不同（`WorkBuddy.exe` / `WorkBuddyAI.exe`），用户数据分别落在 `%APPDATA%\WorkBuddy` 与 `%APPDATA%\WorkBuddy AI`，互不干扰。

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
