# CS2 Slim Updater

自动检测 CS2 更新，并在检测到更新后自动重建 CS2 精简服务端核心包（Linux + Windows），
上传到 [cs2-slim-replica](https://github.com/cyqmq/cs2-slim-replica) 的新 Release。

## 架构

```
┌─────────────────────────────┐
│ Detect CS2 Update (每4小时)  │
│  1. SteamCMD 拉取 CS2 appinfo│
│  2. 解析 public_buildid      │
│  3. 与 state/buildid.json 对比│
│  4. 变化？→ 提交状态 + 触发重建 │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ Rebuild Slim Packages        │
│  1. create-release           │
│     → 计算下一个版本号并创建   │
│  2. build-linux   (并行)      │
│     → 下载 depot → extract   │
│       → 清理编号包 → build    │
│       → 打包 tar.gz → 上传    │
│  3. build-windows (并行)      │
│     → 同上 → 打包 zip → 上传  │
│  4. finalize → 标记 Latest   │
└─────────────────────────────┘
```

## 文件结构

```
├── .github/workflows/
│   ├── detect-cs2-update.yml   # 定时检测（每 4 小时）+ 手动触发
│   └── rebuild-slim.yml        # 重建工作流（自动/手动触发）
├── scripts/
│   ├── get-buildid.py          # SteamCMD 获取 CS2 buildid + depot manifest
│   ├── prepare_tools.py        # 下载 DepotDownloader + steamclient 文件
│   └── cleanup_depot.py        # extract 后删除 pak01 编号包，控制磁盘
└── state/
    └── buildid.json            # 上次已知 buildid（检测工作流自动维护）
```

## 前置配置（必须）

### 1. GH_PAT secret

重建工作流需要把 Release 资产上传到 **另一个仓库**（`cyqmq/cs2-slim-replica`），
默认的 `GITHUB_TOKEN` 不能跨仓库，因此必须配置一个 Personal Access Token：

1. 打开 https://github.com/settings/tokens 生成一个 **Classic PAT**
   - 勾选 `repo` 权限（完整仓库读写，含 Releases）
   - 有效期建议 1 年
2. 在本仓库 **Settings → Secrets and variables → Actions** 添加：
   - Name: `GH_PAT`
   - Value: 上面生成的 token

> 检测工作流本身不需要 GH_PAT（只改本仓库状态文件）。
> 缺少 GH_PAT 时，重建工作流会在 `create-release` 步骤报错提示。

### 2. （可选）更大的 Runner

标准 `ubuntu-latest` 磁盘约 14GB，本工作流已通过 extract 后删除编号包把峰值压到
约 12GB。如果构建因磁盘空间失败，可以在仓库 **Settings → Secrets and variables →
Actions → Variables** 添加：

- Name: `BUILD_RUNNER`
- Value: 例如 `ubuntu-latest-4core`（150GB 磁盘，需付费/支持大型 Runner 的计划）

## 使用方法

### 自动检测

无需手动操作。`Detect CS2 Update` 每 4 小时运行一次：

- 首次运行只记录当前 buildid，**不触发重建**；
- 之后检测到 buildid 变化时，自动：
  1. 提交 `state/buildid.json`
  2. 触发 `Rebuild Slim Packages` 工作流
  3. 重建完成后在 cs2-slim-replica 生成新版本 Release（如 v1.3.0）并标记 Latest

### 手动触发重建

Actions 页 → **Rebuild Slim Packages** → Run workflow，可选填 buildid（留空则读状态文件）。

### 手动触发检测

Actions 页 → **Detect CS2 Update** → Run workflow。

## 构建说明

- 使用 [DepotDownloader](https://github.com/SteamRE/DepotDownloader) 下载 depot：
  - `2347770` 共享内容（含地图/材质/模型等）
  - `2347773` Linux 服务端二进制 / `2347771` Windows 服务端二进制
- steamclient 文件来自 Steam 官方客户端更新包：
  - Linux: `bins_sdk_ubuntu12.zip` → `linux64/steamclient.so`
  - Windows: `bins_win32.zip` → `steamclient64.dll` / `tier0_s64.dll` / `vstdlib_s64.dll`
- 构建复用 `cs2-slim-replica` 的 `cs2slim.py`：
  ```
  python cs2slim.py download --config slim.yaml --depot-tool <path>
  python cs2slim.py extract  --config slim.yaml
  python ../scripts/cleanup_depot.py --workdir ./cs2-build   # 省磁盘
  python cs2slim.py build    --config slim.yaml
  python cs2slim.py package  --config slim.yaml --format tar.gz|zip
  ```

## 产物

每次重建在 `cyqmq/cs2-slim-replica` 生成一个新 Release（版本号自动递增，如 v1.3.0）：

- `cs2-slim.tar.gz` — Linux 核心包
- `cs2-slim-win.zip` — Windows 核心包

`releases/latest/download/...` 链接会自动指向最新版本。

## 常见问题

- **重建工作流报错 “未配置 GH_PAT secret”**
  → 按上文添加 GH_PAT。
- **磁盘空间不足**：确认是否运行在标准 runner；设置 `BUILD_RUNNER` 变量换更大 runner。
- **检测工作流报错找不到 public_buildid**
  → SteamCMD 输出可能因网络暂时失败，重试即可；也可手动运行一次检测。
- **重建失败（DepotDownloader 退出码非 0）**
  → Steam CDN 偶发失败，手动重跑 `Rebuild Slim Packages` 即可。