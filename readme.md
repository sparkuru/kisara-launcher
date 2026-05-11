# kisara-launcher

> 给 Minecraft 私服服主的「整合包一键分发」工具链。
> 服主写 toml，玩家双击 zip，剩下的事自动完成。

## 这是什么

**kisara-launcher 不是启动器**，是一个 CLI 工具，把一个 [packwiz](https://packwiz.infra.link/) 整合包仓库**编译**成可以直接发给玩家的 portable 启动器分发包。

玩家拿到的是一个 zip：解压 → 双击 → 启动器自动检查更新 → 增量下载变动的 mod → 进游戏。**整个过程类似 LOL 客户端的体验。**

## 解决什么问题

服主开私服会反复遇到这套流程：

1. 自己改了 mod / 加了 mod / 升了 loader 版本
2. 把整个 client 目录打 zip
3. 写「读我.txt」教玩家怎么装、怎么删旧 mod、怎么放 Java
4. 发群里 → 一半玩家照着搞错 → 答疑

**痛点**：每次更新都要重复 2-4 步。玩家端没有「点一下就好」的体验。

kisara-launcher 把这套流程拆成两个一次性的步骤：

- 服主**第一次**：写一份 packwiz 仓库，跑一次 `kisara-pack build`
- 服主**之后每次**：改 packwiz 仓库，跑一次 `kisara-pack build --push`
- 玩家**永远只做**：双击启动器

## 项目当前状态

历史上（2024 年）曾打算自己写 Electron 启动器，该方向已废弃，归档于 [doc/legacy-design-2024.md](./doc/legacy-design-2024.md)。

## 技术栈与分工

不重复造轮子，每一层都用社区现成的最佳工具：

| 层 | 用什么 | 干什么 |
|---|---|---|
| **包定义** | [packwiz](https://packwiz.infra.link/) | 服主用 toml 描述「这个整合包有哪些 mod、什么 loader、什么版本」 |
| **玩家端启动器** | [HMCL portable](https://hmcl.huangyuhui.net/) 或 [Prism Launcher portable](https://prismlauncher.org/) | 启动 MC、管理 Java、增量更新 mod |
| **玩家端更新引擎** | [packwiz-installer](https://github.com/packwiz/packwiz-installer) | 启动器 pre-launch hook 触发，对比 hash、增量下载 |
| **认证** | 启动器原生 + authlib-injector | Microsoft 正版 / 第三方 Yggdrasil |
| **kisara-pack（本项目）** | Python CLI | 把上面这些东西**自动组装成玩家分发 zip** |

本项目代码只负责最后一步：**组装**。前面四层都是社区项目。

## 服主友好（admin-friendly）

服主只需要维护一个 git 仓库，里面全是 toml：

```bash
my-server-pack/
├── pack.toml           # 包元信息
├── index.toml          # 文件清单（自动生成）
└── mods/
    ├── jei.pw.toml
    ├── worldedit.pw.toml
    └── sodium.pw.toml
```

日常操作只有这几条命令：

```bash
packwiz mr install jei              # 加 mod（从 Modrinth 自动拉 hash）
packwiz mr update jei               # 升级 mod
packwiz remove some-mod             # 删 mod
packwiz refresh                     # 刷新 index

kisara-pack build                   # 一键构建玩家分发 zip + 服务器同步目录
kisara-pack push                    # rsync 到 HTTP server
```

**服主不写 JSON schema，不算 hash，不调试 OAuth，不处理跨平台 Java**。所有这些已经被下游工具解决。

完整 packwiz pack 示例见 `examples/kisara-dream/`

## 用户友好（player-friendly）

玩家拿到的 zip 解压后长这样：

```
kisara-dream-launcher/
├── 启动游戏.bat / 启动游戏.sh        ← 唯一要点的东西
├── HMCL.exe（或 PrismLauncher.exe）
├── jdk-21/                          ← 自带 Java，不用玩家折腾
└── .minecraft/
    ├── packwiz-installer-bootstrap.jar
    └── versions/kisara-dream/...
```

玩家流程：

1. 下载 zip → 解压
2. 双击「启动游戏」
3. 启动器自动跑 packwiz-installer，对比远端 hash，增量下载差异
4. 进登录页 → 进游戏

**玩家不需要**：装 Java、装启动器、导入整合包、删旧 mod、读「读我.txt」。

之后服主每次更新，玩家**什么也不用做**，下次启动自动同步。

## Usage

...

### 安装

```bash
pip install kisara-pack          # 暂定 Python 实现
```

### 服主：从零开始一个整合包

```bash
# 1. 建 packwiz 仓库
mkdir kisara-dream && cd kisara-dream
packwiz init                                   # 交互式配置 MC + loader 版本

# 2. 加 mod
packwiz mr install jei worldedit sodium

# 3. 配 kisara-pack
cat > kisara.toml <<EOF
[launcher]
type = "hmcl"                                  # 或 "prism"
version = "3.6.x"
[jre]
version = "21"
[server]
manifest_url = "https://mc.example.com/dream/"
auth_server = "https://littleskin.cn/api/yggdrasil"   # 可选，authlib-injector
[branding]
name = "Kisara Dream Launcher"
icon = "./assets/icon.ico"
EOF

# 4. 构建
kisara-pack build
# 产出：
#   dist/player/kisara-dream-launcher.zip      ← 发给玩家
#   dist/server/                               ← 上传到 HTTP server

# 5. 发布
kisara-pack push --target user@host:/var/www/mc/dream/
```

### 服主：日常更新

```bash
packwiz mr update --all
kisara-pack build && kisara-pack push
```

### 玩家

下载 zip → 解压 → 双击「启动游戏」。

## 项目结构

```
kisara-launcher/
├── readme.md
├── doc/
│   ├── legacy-design-2024.md            归档：2024 年的 Electron 方案
│   ├── dev-doc.md                       早期技术探索记录
│   └── test.md                          测试方案
├── src/
│   └── kisara_pack/
│       ├── __main__.py                  CLI 入口
│       ├── packwiz_reader.py            读 packwiz pack.toml / index.toml
│       ├── launcher_fetcher.py          下载 HMCL / Prism portable
│       ├── jre_fetcher.py               下载 Adoptium JRE
│       ├── manifest_builder.py          生成启动器实例配置
│       ├── distro_packer.py             组装玩家分发 zip
│       └── pusher.py                    rsync / scp 推送
├── templates/                           启动器实例配置模板
│   ├── hmcl/
│   └── prism/
├── examples/
│   └── kisara-dream/                    完整的 packwiz pack 示例
├── pyproject.toml
└── license
```

## 路线图

- [ ] CLI 骨架 + 读 packwiz pack
- [ ] HMCL portable 自动下载 + 实例配置生成
- [ ] Prism portable 支持
- [ ] Adoptium JRE 自动适配（按 MC 版本选 JRE 版本）
- [ ] authlib-injector 预配置注入
- [ ] 玩家端 zip 打包 + 启动脚本生成
- [ ] HTTP server 推送（rsync / S3 / OSS）
- [ ] 自定义品牌（图标 / 标题 / 启动画面）
- [ ] 文档站

## 不做的事

明确不在本项目范围内：

- 不自己写启动器 GUI（用 HMCL/Prism）
- 不实现 OAuth 认证（启动器已实现）
- 不托管 mod jar（用 Modrinth/CurseForge CDN，或服主自建镜像）
- 不发明新 schema（用 packwiz 既有格式）

## refer

- [packwiz](https://github.com/packwiz/packwiz) —— 整合包定义与管理
- [packwiz-installer](https://github.com/packwiz/packwiz-installer) —— 玩家端增量更新引擎
- [HMCL](https://github.com/HMCL-dev/HMCL) —— 玩家端启动器（中文社区主流）
- [Prism Launcher](https://prismlauncher.org/) —— 玩家端启动器（国际社区主流）