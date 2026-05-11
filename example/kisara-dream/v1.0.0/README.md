# kisara-dream v1.0.0（初次分发）

## 服主侧文件树

```
v1.0.0/
├── pack.toml                  # 包元信息：名字、版本、MC + loader 版本、index 指针
├── index.toml                 # 全部文件清单 + hash（packwiz 自动生成/刷新）
└── mods/
    ├── jei.pw.toml            # 必装
    ├── worldedit.pw.toml      # 必装
    └── forgematica.pw.toml    # 可选（client 端，默认不勾）
```

## 服主操作命令

```bash
# 第一次建包
packwiz init                                  # 交互式问：名字/MC 版本/loader
packwiz mr install jei                        # 从 Modrinth 装 mod（自动算 hash、写 .pw.toml）
packwiz mr install worldedit
packwiz url add forgematica https://github.com/.../forgematica-0.1.3.jar
packwiz refresh                               # 重算 index.toml

# 提交
git add . && git commit -m "v1.0.0" && git push
```

## 服主侧上传到 HTTP server

整个 `v1.0.0/` 目录直接 rsync 到 `https://domain.com/mc/kisara-dream/`，
玩家端 URL = `https://domain.com/mc/kisara-dream/pack.toml`

## 玩家侧体验（用 packwiz-installer + Prism/HMCL）

1. 服主把 **Prism portable + 配好的 instance.cfg + packwiz-installer-bootstrap.jar** 打包
   发给玩家（zip 解压即用，~50MB）
2. 玩家解压双击 prismlauncher.exe
3. 启动器 pre-launch 钩子自动跑 `java -jar packwiz-installer-bootstrap.jar <pack URL>`
4. installer 对比本地 ↔ 远端 hash，下载缺失/变动的 mod；弹窗让玩家勾选 optional 的 forgematica
5. 玩家点确认 → 进游戏

## 服主端首次分发包结构（发给玩家的 zip）

```
kisara-dream-launcher.zip
├── prismlauncher.exe                  # 或 HMCL.exe
├── instances/
│   └── kisara-dream/
│       ├── instance.cfg               # 预配 pre-launch: java -jar bootstrap.jar URL
│       ├── packwiz-installer-bootstrap.jar
│       └── .minecraft/                # 空目录，首次启动由 installer 填充
└── 启动游戏.bat                       # = prismlauncher.exe（一键）
```

玩家除了双击 `启动游戏.bat`，什么也不用做。
