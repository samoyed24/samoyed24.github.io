+++
date = '2026-08-10T00:10:00+08:00'
draft = false
title = 'MegaFunPack 服务器进入指南'
description = 'Minecraft 1.20.1 整合包 MegaFunPack 的下载、安装与连接说明'
+++

欢迎加入 **MegaFunPack** 服务器！这是一个融合 **科技 · 冒险 · 魔法 · 建筑** 的 MC 1.20.1 整合包。本文介绍如何从零开始进入服务器。

---

## 一、环境要求

- **PCL2**（启动器，支持 Windows）
- 电脑内存建议 **8G 以上**（整合包含 78 个模组）
- 已安装 Java 17 或 21

## 二、下载整合包

客户端整合包（模组已全部内置，无需联网下载）：

> **下载地址**：<https://dl.portcloud.online/mcserver/MegaFunPack-client-full.zip>

文件较大（约 380MB），请耐心等待。

## 三、安装（PCL2）

1. 打开 **PCL2**，进入左侧「版本列表」
2. 点击「**安装整合包**」，选择刚下载的 `MegaFunPack-client-full.zip`（或直接把 zip 拖进 PCL2 窗口）
3. PCL2 会自动创建 `MegaFunPack` 实例，并安装 **Forge 47.4.22** + 解压全部 78 个模组
4. 首次启动前，在「设置 → 启动 → 内存设置」把内存拉到 **6G~8G**
5. 选择 `MegaFunPack` 版本启动（首次加载较久，属正常）

## 四、连接服务器

- **服务器地址**：`minecraft.portcloud.online`
- **端口**：默认（25565，无需填写）

进入游戏 → 多人游戏 → 添加服务器 → 地址填 `minecraft.portcloud.online` → 加入。

## 五、账号登录（DAuth）

服务器为**离线模式 + 密码登录**，第一次进入需注册：

1. 进入游戏后会自动弹出**登录界面**
2. 首次：点击「创建账号」，设置**任意字符密码**（支持大小写/数字/符号），密码显示为 `****`
3. 之后：输入密码登录即可
4. 注意：密码连错 3 次会被踢出服务器

## 六、服务器模组列表（78 个）

| 分类 | 模组 | 说明 |
|---|---|---|
| **加载器** | Forge 47.4.22 | 模组加载器 |
| **性能优化** | Embeddium | 渲染优化，大幅提升 FPS |
| | Canary | 服务端/物理运算优化 |
| | FerriteCore | 内存优化 |
| | ModernFix | 内存 / 启动速度优化 |
| | ImmediatelyFast | 减少卡顿 |
| | MemoryLeakFix | 修复内存泄漏 |
| **科技 / 自动化** | Create 机械动力 | 机械传动自动化 |
| | Create: Steam 'n' Rails | 火车 / 轨道扩展 |
| | Create: Diesel Generators | 柴油发电机 |
| | Mekanism | 工业 / 核电科技线 |
| | AE2 应用能源 | 存储与物流 |
| | Functional Storage | 存储抽屉 |
| | Pipez | 管道运输 |
| | Iron Chests | 更多容量箱子 |
| | Tech Reborn | 科技模组 |
| **冒险 / 探索 / 维度** | Terralith | 全新地形群系 |
| | Alex's Mobs | 大量新生物 |
| | Alex's Caves | 全新维度 |
| | The Aether | 天堂维度 |
| | Twilight Forest | 暮色森林维度 |
| | TF Dungeons & Villages | 暮色森林扩展 |
| | YUNG's Better Dungeons | 更精致的地牢 |
| | YUNG's Better Strongholds | 更精致的要塞 |
| | YUNG's Better Mineshafts | 更精致的废弃矿洞 |
| | YUNG's Better Ocean Monuments | 更精致的海底神殿 |
| | YUNG's Better Nether Fortresses | 更精致的下界要塞 |
| | YUNG's Better Witch Huts | 更精致的女巫小屋 |
| | YUNG's Better Jungle Temples | 更精致的丛林神殿 |
| | YUNG's Better Desert Temples | 更精致的沙漠神殿 |
| **魔法** | Botania | 植物魔法 |
| | Iron's Spells & Spellbooks | 铁之法术 |
| **建筑** | Chipped | 每种方块多种变体 |
| | Handcrafted | 家具 |
| | Rechiseled | 方块切割组合 |
| | Supplementaries | 装饰 + 实用工具 |
| | Decorative Blocks | 装饰方块 |
| | Cooking for Blockheads | 厨房料理 |
| | Amendments | 原版方块增强 |
| **实用 QoL** | JEI | 配方查看 |
| | Jade | 准星指向信息 |
| | AppleSkin | 显示饱食度 / 饱和值 |
| | Xaero's Minimap | 小地图 |
| | Xaero's World Map | 世界地图 |
| | Inventory Profiles Next | 一键整理背包 |
| | Mouse Tweaks | 鼠标拖拽整理 |
| | ShulkerBoxTooltip | 潜影盒内容预览 |
| | No Chat Reports | 移除聊天签名 |
| | Controlling | 按键搜索 |
| | Waystones | 传送石碑 |
| **沉浸 / 联机** | Sound Physics Remastered | 真实音效 |
| | Ambient Sounds | 环境音效 |
| | Simple Voice Chat | 联机语音（UDP 24454） |
| | DAuth | 密码登录 |

---

## 七、小贴士

- **小地图 / 地图**：按 `M` 键打开世界地图，`H` 键小地图设置
- **查看配方**：按 `E` 打开背包，鼠标指向任意物品查看合成表
- **语音联机**：需客户端同样启用 Simple Voice Chat，进服自动连接
- 若连接失败，请确认客户端模组与服务端一致（78 个），否则会被服务器拒绝

玩得开心！
