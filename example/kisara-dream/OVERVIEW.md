# packwiz pack 示例总览

> 注意：所有 hash 是手写占位（演示用）。真实 packwiz 会自动算 sha256/sha512，

## 目录树

```
/tmp/tmp/kisara-dream/
├── OVERVIEW.md
├── v1.0.0/                              ← 初次分发
│   ├── README.md                        服主操作说明
│   ├── pack.toml                        包元信息（version=1.0.0）
│   ├── index.toml                       文件清单 + hash
│   └── mods/
│       ├── jei.pw.toml                  必装
│       ├── worldedit.pw.toml            必装
│       └── forgematica.pw.toml          可选（client-only）
└── v1.0.1/                              ← 更新分发
    ├── README.md                        服主操作说明 + 变更日志
    ├── pack.toml                        version=1.0.1
    ├── index.toml                       
    └── mods/
        ├── jei.pw.toml                  ← 版本升级
        ├── worldedit.pw.toml            ← 没变
        └── sodium.pw.toml               ← 新增
        # forgematica.pw.toml 删除
```

## packwiz 核心抽象（三层）

1. **`pack.toml`**：整个包的「身份证」，含 MC + loader 版本、指向 index 的指针。
   玩家端先拉这个，看到 version 变了就知道要更新。
2. **`index.toml`**：清单，记录每个被 packwiz 管理的文件及其 hash。
   增量更新的关键 —— 对比这份清单就知道哪些文件要重下。
3. **`mods/<name>.pw.toml`**（元文件）：每个 mod 的真实下载地址 + hash + 更新源。
   注意：**mod 的 jar 本身不在 packwiz 仓库里**，只存元数据。
   仓库可以小到几百 KB。整个 git 化、git diff 看得懂。

## 服主端 vs 玩家端的资源边界

| 谁存什么 | 服主仓库（git） | HTTP server | 玩家本地 |
|---|---|---|---|
| `pack.toml` / `index.toml` / `*.pw.toml` | ✓ | ✓ | ✓（packwiz-installer 拉的） |
| mod 的 jar 本体 | ✗（不存） | ✗（除非自建镜像） | ✓（installer 从 Modrinth/CF/直链下载到本地） |
| 配置文件 (`config/*`) | ✓ | ✓ | ✓ |
| 整合包 zip | ✗ | ✗ | ✗（不需要） |

**关键**：你的 HTTP server 只托管 `pack.toml` + `index.toml` + `*.pw.toml` + `config/`，
mod 的 jar 还是从 Modrinth/CurseForge 的 CDN 下，省你带宽。
如果你不信 CDN，可以在 `[download]` 里指向自己的镜像，**支持混用**。

## 玩家端工作流

```
玩家双击启动器
     │
     ▼
Prism/HMCL pre-launch hook：
   java -jar packwiz-installer-bootstrap.jar https://domain.com/mc/kisara-dream/pack.toml
     │
     ▼
bootstrap 自动下载主 installer jar（缓存复用）
     │
     ▼
installer 拉 pack.toml → 看 version
     │
     ├─ 与本地 pack.toml.local 比对：相同 → 跳过更新，直接启动
     │
     └─ 不同 → 拉 index.toml → 拉所有 *.pw.toml
                │
                ▼
        逐文件对比 hash：
            - 一致 → 跳过
            - 不一致或缺失 → 从 [download].url 下载到本地、校验 hash
            - optional 且未勾选 → 跳过/删除本地副本
            - 远端没了 → 删本地（仅限 packwiz 管理范围）
                │
                ▼
        弹窗汇报：「新增 1 / 更新 1 / 删除 1」→ 关闭 → MC 启动
```

## 对照你 readme 里的 `mods_config`

2024 年设计的：

```json
{
  'name': '[创世神]-worldedit-v7.3.0.jar',
  'path': 'games/kisara-dream/mods/[创世神]-worldedit-v7.3.0.jar',
  'url': 'https://cdn.modrinth.com/.../worldedit.jar',
  'hash': '54321',
  'type': 'required'
}
```

packwiz 等价物（worldedit.pw.toml）：

```toml
name = "WorldEdit"
filename = "worldedit-mod-7.2.15.jar"
side = "both"

[download]
url = "https://cdn.modrinth.com/.../worldedit.jar"
hash-format = "sha512"
hash = "..."

# required 是默认；optional 用 [option] 块表示
```

**字段几乎一对一**。你 2024 年没看到 packwiz 而独立想出了一样的 schema，
说明这个设计是正确的，只是已经有人做了完整实现。

## 配套需要的玩家端打包（这部分是你要写的 CLI 的活）

```
kisara-dream-launcher-v1.0.0.zip
├── prismlauncher.exe                       (~50 MB)
├── jdk-21/                                 (~200 MB，可选自带；也可让 Prism 首次自动装)
├── instances/
│   └── kisara-dream/
│       ├── instance.cfg                    预配 pre-launch hook
│       ├── mmc-pack.json                   MC + Forge 版本声明
│       └── .minecraft/
│           └── packwiz-installer-bootstrap.jar
└── 启动游戏.bat                            ← 一行：start prismlauncher.exe
```

**CLI 要做的事**：

1. 读 packwiz pack 里的 `pack.toml`，提取 MC + loader 版本
2. 生成 Prism 的 `mmc-pack.json` + `instance.cfg`
3. 下载 Prism portable + JDK
4. 组装目录、塞 bootstrap.jar、写 pre-launch hook
5. 打 zip