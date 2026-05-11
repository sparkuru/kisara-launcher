# 归档：kisara-launcher 2024 年设计

> 本文档归档 2024 年初版设计。当时打算「自己写一个 Electron + Vue 启动器」。
> 在 2026 年重新评估后，该方案已废弃，详见根目录 [readme.md](../readme.md) 的新方向。
> 保留此文档仅为追溯历史决策。

## 一、原本考察过的技术栈

forks 自 [Shanwer/NsisoLauncher](https://github.com/Shanwer/NsisoLauncher.git)（root：[Coloryr/NsisoLauncher-1](https://github.com/Coloryr/NsisoLauncher-1.git)，另一版本：[Coloryr/ColorMC](https://github.com/Coloryr/ColorMC.git)）。原工程使用 C# + XAML，但 2024 年决定重写，尝试过以下技术组合：

| 前端                           | 后端                                                | 交互                                                         | 评估结论                                                     |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| electron + vue                 | cpp 编译为可执行文件，httplib 开放在特定端口        | axios                                                        | 不方便打包成单个应用                                         |
| electron + vue                 | cpp                                                 | ffi-napi + node-gyp 编译为 `*.node`                          | 跨平台差异化调整麻烦                                         |
| electron + vue                 | cpp                                                 | Emscripten 编译为 `*.wasm`                                   | Windows 下 Emscripten portable 化麻烦                        |
| **electron + vue + vite + js** | —                                                   | —                                                            | **一把梭，如无必要、勿增实体**（最终选择，但整个方向后来被放弃）|

详细搭建过程见同目录的 [dev-doc.md](./dev-doc.md)。

## 二、原本计划的启动器功能矩阵

-   **首页**：公告 + 账号密码登录 + 设置入口 + 版本隔离的 mod 包选择
-   **账号认证**：官方/第三方 OAuth2、authlib-injector 地址配置、注册一键直达、皮肤头像获取
-   **游戏启动设置**：全局 default + 单包特定 Java、窗口名、版本名
-   **下载设置**：源选择、代理、MC 资源、JRE 路径、mods 更新
-   **后端**：调系统 cmd 启动 `java -jar`，详细 log
-   **(可选)** mod 列表页带百科介绍 + 链接

## 三、原本计划的 src-tree（C# 风格残留）

-   **gui**：前端
    -   Config / Resource / Themes / Utils
-   **core**：后端
    -   Authenticator / Component / Config / LaunchException / Modules / Nbt / Net / User / Util
-   **coretest**：后端测试
    -   LauncherMetaApiTest / ServerInfoTest

## 四、为什么放弃这个方向

1. **核心需求已被现成方案覆盖**：
    - 「服主自定义整合包 + 自动更新」→ [packwiz](https://packwiz.infra.link/) 已是事实标准
    - 启动器 GUI 部分 → [HMCL](https://hmcl.huangyuhui.net/) / [Prism Launcher](https://prismlauncher.org/) 已经做得非常好
    - authlib-injector → HMCL 原生支持
    - JRE 管理、Java 版本隔离、增量更新、hash 校验 → 上述工具均已实现
2. **Electron 在 2026 年不再适合启动器场景**：bundle 150MB+，启动慢，Tauri 2 已成熟但仍不必要
3. **维护成本不对等**：自己造一个启动器要长期跟进 MC 协议变化、loader 变化、各平台 OAuth 流程；而拼装现成工具的成本只有写一个 ~200 行 CLI

新方向不再造启动器，转为**写一个 CLI 工具，把 packwiz pack 编译成「玩家一键解压即用」的 portable 启动器分发包**。详见 [readme.md](../readme.md)。

## 五、licenses & thanks（原方案的依赖致谢）

新方案不再依赖以下库（这些是 C# 启动器时代的依赖），归档于此：

-   [BMCLAPI](https://bmclapidoc.bangbang93.com/)
-   [MahApps/MahApps.Metro](https://github.com/MahApps/MahApps.Metro)
-   [MahApps/MahApps.Metro.IconPacks](https://github.com/MahApps/MahApps.Metro.IconPacks)
-   [JamesNK/Newtonsoft.Json](https://github.com/JamesNK/Newtonsoft.Json)
-   [Fody/Fody](https://github.com/Fody/Fody)
-   [Fody/Costura](https://github.com/Fody/Costura)
-   [Live-Charts/Live-Charts](https://github.com/Live-Charts/Live-Charts)
-   [icsharpcode/SharpZipLib](https://github.com/icsharpcode/SharpZipLib)
-   [hawezo/MojangSharp](https://github.com/hawezo/MojangSharp)
-   [ghuntley/Heijden.Dns](https://github.com/ghuntley/Heijden.Dns)
-   [Minecraft-Console-Client](https://github.com/ORelio/Minecraft-Console-Client)
-   [cyotek/Cyotek.Data.Nbt](https://github.com/cyotek/Cyotek.Data.Nbt)
