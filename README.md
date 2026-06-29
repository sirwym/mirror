# TurboWarp Editor 构建包仓库

> 本仓库是一个 **离线构建包（Build Package）仓库**，通过 GitHub Actions 周期性构建并打包 [TurboWarp](https://turbowarp.org/) 图形化编辑器的离线产物，供 [LearnHouse](https://github.com/sirwym/learnhouse-cn) 平台内嵌使用。

## 仓库定位

| 项目 | 说明 |
|------|------|
| 名称 | TurboWarp Editor 构建包 |
| 用途 | 周期性构建可离线运行的 TurboWarp Editor 静态包 |
| 消费方 | [LearnHouse](https://github.com/sirwym/learnhouse-cn) `apps/scratch` |
| 部署方式 | GitHub Pages + GitHub Actions Artifact + GitHub Release |
| 输出物 | `scratch-gui/build/` 静态文件 + `turbowarp-build.zip` 离线包 |

## 为什么需要这个仓库？

[LearnHouse](https://github.com/sirwym/learnhouse-cn) 是一个面向中小学生（K12 / 3-9 年级）的编程教育 SaaS 平台，平台内的"编程实验室"模块需要嵌入一个**可离线运行、不依赖第三方 CDN** 的图形化 Scratch 编辑器，让学生：

- 打开 `*.sb3` 模板进行本地练习
- 自由创作并把作品保存在本地（不上传服务器）
- 在练习模式下脱机使用编辑器

直接拉取上游 [TurboWarp/scratch-gui](https://github.com/TurboWarp/scratch-gui) 仓库的构建产物存在两个问题：

1. **体积大 / 网络受限** —— 上游构建产物在国内拉取不稳定，学生端首屏慢
2. **需要定制化** —— 需要替换 trampoline 跳转地址、移除 PWA manifest、强制 `noindex` 等

本仓库即用于解决上述问题：

- 每周由 GitHub Actions 自动拉取上游 `TurboWarp/scratch-gui` 最新代码并构建
- 在构建产物上应用自定义补丁（见 [`patch.py`](./patch.py)）
- 打成 ZIP 离线包发布到 GitHub Release
- 同时部署到 GitHub Pages，方便直接以 URL 方式拉取

## 与 LearnHouse 的消费关系

```
┌─────────────────────────┐    下载/拉取     ┌──────────────────────────────┐
│  本仓库 (mirror)        │ ───────────────▶ │  LearnHouse apps/scratch     │
│  GitHub Actions 构建     │   离线 ZIP 包    │  download_turbowarp.sh       │
│  turbowarp-build.zip    │   或 GitHub      │  ↓                            │
│  GitHub Pages 部署       │   Pages URL     │  public/ (解压到静态目录)    │
└─────────────────────────┘                  │  ↓                            │
                                              │  Bun.serve() 自托管 (5800)   │
                                              │  ↓                            │
                                              │  LearnHouse Web 端 iframe 嵌入│
                                              └──────────────────────────────┘
```

在 LearnHouse 仓库中执行：

```bash
# 方式一：直接消费 GitHub Pages 部署
cd apps/scratch
bun run download
# 默认从 https://sirwym.github.io/mirror/ 镜像下载

# 方式二：消费本仓库的 Release ZIP
cd apps/scratch
TURBOWARP_SOURCE=/path/to/turbowarp-build.zip bun run download

# 方式三：消费本仓库的 Actions artifact
cd apps/scratch
TURBOWARP_SOURCE=/path/to/artifact.zip bun run download

# 方式四：消费本地构建产物
cd apps/scratch
TURBOWARP_SOURCE=/path/to/scratch-gui/build bun run download
```

下载完成后，在 LearnHouse 启动 scratch 服务：

```bash
bun run dev    # 默认 5800 端口，可在 .env 用 SCRATCH_PORT 覆盖
```

前端 `apps/web` 通过 iframe 访问 `http://localhost:5800/` 即可嵌入编辑器。

## 构建产物

| 产物 | 路径 / 名称 | 用途 |
|------|------------|------|
| 静态文件目录 | `scratch-gui/build/` | 部署到 GitHub Pages，提供 URL 访问 |
| 离线 ZIP 包 | `turbowarp-build.zip` | 上传为 GitHub Actions Artifact，30 天保留 |
| GitHub Release | `build-YYYYMMDDHHMMSS` Tag | 手动触发时创建，长期保存离线包 |

ZIP 包内容与 `scratch-gui/build/` 完全一致，仅多一层 zip 包装，可直接解压到任何静态服务器。

## 构建流程

构建由 [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml) 编排：

```
Checkout self ──▶ Checkout TurboWarp/scratch-gui ──▶ Install Node 22
                                                       │
                                                       ▼
                                          npm ci + npm run build
                                          (ROUTING_STYLE=hash)
                                                       │
                                                       ▼
                                              patch.py 应用定制
                                                       │
                                                       ▼
                              ┌────────────────────────┼────────────────────────┐
                              ▼                        ▼                        ▼
                    Deploy to GitHub Pages    Upload ZIP Artifact    (Tag/Manual) Create Release
```

### 触发方式

| 触发条件 | 行为 |
|---------|------|
| `push` 到 `main` | 构建 + 部署到 GitHub Pages |
| 每周二 18:52 (UTC) | 自动构建（拉取上游最新代码） |
| **打任意 tag**（如 `v1.0.0`、`build-20260630`） | 构建 + 部署 + 创建带 ZIP 的 GitHub Release，Release tag 即你打的 tag 名 |
| 手动 `workflow_dispatch` | 构建 + 部署 + 创建带 ZIP 的 GitHub Release，Release tag 自动生成 `build-YYYYMMDDHHMMSS` |

**发布新 Release 的标准流程：**

```bash
# 1. 在 main 分支上打个 tag
git tag v1.0.0
git push origin v1.0.0

# 2. 等待 Actions 跑完（约 5-10 分钟）
# 3. 在 Releases 页面即可看到 TurboWarp Offline Build v1.0.0
```

## 自定义补丁

`patch.py` 在上游 `scratch-gui` 基础上做了以下改动：

| 改动 | 目的 |
|------|------|
| `<meta name="robots" content="noindex">` | 避免被搜索引擎索引 |
| 移除 `manifest.webmanifest` 引用 | 关闭 PWA / Add to Home Screen |
| `https://trampoline.turbowarp.org` → `https://trampoline.turbowarp.xyz` | 跳转走国内镜像 |
| 删除 `sw.js` / `fullscreen.html` / 原始 `index.html` | 移除 Service Worker、Player 入口等非必要文件 |
| `editor.html` → `index.html` | 把编辑器作为默认入口 |
| 写入 `robots.txt` | 禁止爬虫 |

如需调整补丁，编辑 [`patch.py`](./patch.py) 并重新触发 workflow 即可。

## 本地构建

```bash
# 1. 克隆本仓库
git clone https://github.com/sirwym/mirror.git
cd mirror

# 2. 克隆上游 scratch-gui
git clone https://github.com/TurboWarp/scratch-gui.git

# 3. 构建上游
cd scratch-gui
npm ci
NODE_ENV=production ROUTING_STYLE=hash npm run build

# 4. 应用自定义补丁
cd ..
python3 patch.py

# 5. 产物在 scratch-gui/build/，可打成 zip
cd scratch-gui/build
zip -r ../../turbowarp-build.zip ./*
```

## 在 LearnHouse 中集成新构建包

当你发布了一个新的 `turbowarp-build.zip` 后，LearnHouse 端可以这样升级：

```bash
# 在 LearnHouse 仓库根目录
cd apps/scratch

# 拉取最新 Release 包
gh release download --repo sirwym/mirror --pattern 'turbowarp-build.zip'
mv turbowarp-build.zip /tmp/

# 解压到 public/
TURBOWARP_SOURCE=/tmp/turbowarp-build.zip bun run download

# 重启 scratch 服务
pm2 restart learnhouse-scratch
```

## 目录结构

```
mirror/
├── .github/
│   └── workflows/
│       └── deploy.yml          # 构建 / 部署 / Release 工作流
├── patch.py                    # 上游构建产物的定制脚本
├── robots.txt                  # 写入构建产物的 robots.txt
├── LICENSE                     # MIT（构建脚本许可证）
└── README.md                   # 本文件
```

> 说明：`scratch-gui/` 在 `.gitignore` 中排除，构建时由 Actions 临时拉取。

## 维护说明

- **升级上游**：默认每周二 18:52 UTC 自动拉取 `TurboWarp/scratch-gui` 的 `HEAD`，无需手动干预
- **强制重新构建**：在 GitHub Actions 页面 `Actions` → `Deploy and Release` → `Run workflow` → 选择 `main` 分支运行
- **指定上游版本**：在 `deploy.yml` 的 `Checkout scratch-gui` 步骤加 `ref: <commit-sha>` 即可锁定版本
- **包大小监控**：每次构建后查看 Actions Artifact 大小，正常情况下构建产物在 6-10 MB

## 许可证

- 本仓库的构建脚本（`patch.py`、workflow 配置）使用 **MIT 协议**（见 [LICENSE](./LICENSE)）
- 生成的 `turbowarp-build.zip` 内含 [TurboWarp/scratch-gui](https://github.com/TurboWarp/scratch-gui) 的编译产物，遵循其 **GPL 3.0** 协议
- 部署到 LearnHouse 后由 `apps/scratch` 的 Bun 静态文件服务器对外提供（同样遵循 GPL 3.0）
