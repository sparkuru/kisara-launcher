# kisara-dream v1.0.1（更新分发）

## 相对 v1.0.0 的变更

| 文件 | 变化 | 说明 |
|---|---|---|
| `pack.toml` | version `1.0.0 → 1.0.1`，hash 变了 | 触发玩家端检测到更新 |
| `index.toml` | hash 变了，少一行多一行 | mod 列表变了 |
| `mods/jei.pw.toml` | jei 版本 15.3 → 15.9，hash 变 | 玩家端会重下 jei |
| `mods/worldedit.pw.toml` | **完全没变** | 玩家端 hash 一致，跳过 |
| `mods/forgematica.pw.toml` | **被删除** | 玩家端会删掉这个 mod |
| `mods/sodium.pw.toml` | **新增** | 玩家端会下载（client side） |

## 服主操作命令

```bash
packwiz mr update jei                  # 拉 modrinth 最新版，重算 hash
packwiz remove forgematica             # 不再分发投影
packwiz mr install sodium              # 新增 sodium
packwiz refresh                        # 重算 index.toml + pack.toml 的 hash

git add . && git commit -m "v1.0.1" && git push

# 上传：直接覆盖 server 上的 v1.0.0 目录即可
rsync -av --delete ./ user@server:/var/www/mc/kisara-dream/
```

## 玩家侧体验

1. 玩家双击 `启动游戏.bat`，无需任何操作
2. Prism/HMCL 的 pre-launch 钩子跑 `packwiz-installer-bootstrap.jar`
3. installer 拉 `pack.toml`，看到 version 1.0.1 + hash 不同，进入更新流程
4. 对每个 mod 检查本地 hash：
   - jei：本地是 15.3，远端 15.9 → 删本地、下新的
   - worldedit：hash 一致 → 跳过（**关键：增量更新，不重下整个包**）
   - forgematica：远端没了 → 删本地（除非玩家在 `manual-mods` 目录自留）
   - sodium：远端有、本地没 → 下载
5. installer 关闭 → MC 启动

整个过程对玩家而言只是「启动器多转了 3 秒进度条」。

## 增量更新的精确语义（重要）

packwiz-installer 不删玩家自己加到 `manual-mods/` 的 mod。
只对 packwiz 自己管理的 `mods/` 做强同步。

这一点对应你 readme 里写的 `extra_mods` 文件夹 —— packwiz 默认就这么干。
