# iOSBuilder

基于 GitHub Actions 的 iOS IPA 自动构建工具，无需本地 Mac，从任意引擎导出的 Xcode 工程自动编译、伪造签名、打包 IPA。

---

## 工作原理

```
引擎导出 Xcode 工程.zip → GitHub Releases
         ↓
GitHub Actions 启动 macOS 虚拟机
         ↓
download.sh 下载 zip → 解压 → 得到 Xcode 工程
         ↓
xcodebuild clean archive → 编译出 .app
         ↓
ldid 伪造签名（免开发者证书）
         ↓
打包 .ipa → 上传到 Actions Artifact
         ↓
Actions 页面下载 .ipa
```

---

## 切换引擎

本工作流编译的是 **Xcode 工程**，不是特定引擎的产物。只要你的引擎能导出 iOS 的 Xcode 工程（`.xcodeproj`/`.xcworkspace`），就能用。

| 引擎 | 导出方式 | 需修改的地方 |
|------|----------|-------------|
| **Unity**（当前） | File → Build Settings → iOS → Export | 默认即可 |
| **Unreal** | Platforms → iOS → Package Project | `scheme` 改为项目名，产物通常是 `.xcworkspace` |
| **Godot** | Export → iOS → Export | `scheme` 改为项目名 |
| **Cocos2d-x** | 自带 Xcode 工程 | `scheme` 改为项目名 |
| **原生 Xcode 项目** | 直接在 Xcode 中创建 | `scheme` 改为 target 名 |

**需要改什么：**
1. `build.yml` 的 `scheme` 环境变量 — 改为你 Xcode 工程的 Scheme 名称
2. `download.sh` 的下载链接 — 改为你引擎导出的压缩包
3. 压缩包内容 — 确保根目录包含 `.xcodeproj` 或 `.xcworkspace`

工作流会自动识别是 `.xcworkspace` 还是 `.xcodeproj`，无需手动配置。

---

## 快速开始

### 1. 准备 Xcode 工程

在你使用的引擎中导出 iOS Xcode 工程，将所有文件打包为一个 zip。

**以 Unity 为例：**
- **Il2Cpp 后端：** 主工程 + Il2CppOutputProject 分开打包，参考 `download.sh`
- **Mono 后端：** 一个完整工程包即可，参考 `download2.sh`

### 2. 上传压缩包到 Releases

进入你的 GitHub 仓库 → **Releases** → **Create a new release** → 上传 zip → 发布。

### 3. 修改 download 脚本

把下载链接换成你自己的 Release 地址。

如果用 `download.sh`：

```bash
# 修改前
curl -LO https://github.com/StArrayJaN/iOSBuilder/releases/download/1.0/MainProject.zip
curl -LO https://github.com/StArrayJaN/iOSBuilder/releases/download/1.0/Il2CppOutputProject.zip

# 修改后
curl -LO https://github.com/你的用户名/你的仓库/releases/download/0.1/YourProject.zip
```

如果用 `download2.sh`：

```bash
# 用 sed 替换版本号
sed -i "s|1.0|0.1|g" download2.sh

# 或替换整行下载链接
sed -i "5d" download2.sh
sed -i "5icurl -LO https://github.com/你的用户名/你的仓库/releases/download/0.1/XcodeProject.zip" download2.sh
```

### 4. 触发构建

| 方式 | 操作 |
|------|------|
| 手动触发 | **Actions** → **Build IPA directly** → **Run workflow** |
| 推送触发 | `git push` 到仓库自动运行 |
| Release 触发 | 创建或编辑 Release 时自动运行 |

### 5. 下载 IPA

构建完成后，在 Actions 运行记录的 **Artifacts** 区域下载 `.ipa` 文件。

---

## 文件说明

### `.github/workflows/build.yml`

核心 CI 流程，10 个步骤：

| 步骤 | 作用 |
|------|------|
| setup-xcode | 安装指定版本的 Xcode |
| Checkout | 拉取仓库代码 |
| Run download script | 执行 `download.sh` 或 `download2.sh` |
| Set Scheme | 自动检测或使用预设的 Xcode Scheme |
| Set filetype_parameter | 自动识别 `.xcworkspace` 或 `.xcodeproj` |
| Set file_to_build | 定位工程文件 |
| Build and Archive | `xcodebuild clean archive` 编译 |
| Install ldid | 安装伪造签名工具 |
| Fakesign | 对所有 Framework 和 .app 执行伪造签名 |
| Create IPA | 打包为 `.ipa` 并上传 Artifact |

### `download.sh` — Il2Cpp 版

适用于 Unity + Il2Cpp 后端：

- 下载 `MainProject.zip`（主工程）
- 下载 `Il2CppOutputProject.zip`（Il2Cpp 输出）
- 下载 `Info.plist`（覆盖默认配置）
- 权限设置：`usymtool`、`usymtoolarm64`

### `download2.sh` — Mono 版

适用于 Unity Mono 后端，只需一个完整的 `XcodeProject.zip`。

---

## 从原仓库同步更新

如果你的仓库 fork 自 [StArraySharp/iOSBuilder](https://github.com/StArraySharp/iOSBuilder) 或 [StArrayJaN/iOSBuilder](https://github.com/StArrayJaN/iOSBuilder)，需要拉取上游更新时：

```bash
# 添加原仓库为上游
git remote add upstream https://github.com/StArrayJaN/iOSBuilder.git

# 拉取并合并（允许无关历史）
git pull upstream main --allow-unrelated-histories

# 如果有冲突，手动解决后提交
git add .
git commit -m "合并上游更新"
git push
```

---

## 清理 Git 历史中的大文件

如果不小心把大文件（如 `.zip`）提交到了 Git 历史中，会导致仓库体积膨胀。使用 `git filter-repo` 清理：

```bash
# 安装
pip install git-filter-repo

# 在你的仓库目录下执行（需要是 fresh clone）
# 移除指定路径的所有历史记录
git filter-repo --path '.github/workflows/XcodeProject.zip' --invert-paths --force

# 强制推送覆盖远程历史
git push origin --force --all
git push origin --force --tags
```

> **注意：** `git filter-repo` 需要在 fresh clone 上执行，否则会报 `Aborting: Refusing to destructively overwrite repo history`。
> 如果遇到此错误，加 `--force` 参数即可。

---

## 便捷推送脚本

如果网络不稳定导致 `git push` 经常失败，可以创建一个重试脚本：

```bash
#!/bin/bash
echo "# 微小更改 $(date)" >> download2.sh
git add download2.sh
git commit -m "添加注释到 download2.sh"
until git push; do
    echo "推送失败，200ms后重试..."
    sleep 0.2
done
```

保存为 `push.sh`，以后直接 `bash push.sh` 即可自动重试推送。

---

## 常见问题

### Q：生成的 IPA 能装到 iPhone 上吗？

不能直接安装。使用 `ldid` 伪造签名，没有 Apple 开发者证书。

| 用途 | 做法 |
|------|------|
| 越狱手机 / TrollStore | 直接安装 |
| 真机调试 | 用 `codesign` 重新签名开发者证书 |
| App Store 上架 | 需 Xcode + 付费账号，不适用本项目 |

### Q：构建失败排查

| 错误 | 原因 | 解决 |
|------|------|------|
| `未找到 .app 文件` | Scheme 名称不对 | 检查 `build.yml` 中的 `scheme` 配置 |
| `未找到 Unity-iPhone.xcodeproj` | 压缩包结构不对 | 确认 zip 解压后根目录有 `.xcodeproj` |
| `curl: (22) ... 404` | 下载链接错误 | 检查 Release 链接和版本号 |
| `xcodebuild error` | 编译错误 | 检查 Unity 导出是否完整 |
| `filter-repo Aborting` | 不是 fresh clone | 加 `--force` 参数 |

### Q：如何选择 download.sh 还是 download2.sh？

| | Mono | Il2Cpp |
|--|------|--------|
| 脚本 | `download2.sh` | `download.sh` |
| 压缩包 | 1 个完整 zip | 主工程 + Il2Cpp 输出分开 |
| Unity 设置 | Scripting Backend → Mono | Scripting Backend → IL2CPP |

---

## 免责声明

本工具仅供学习和个人研究使用。使用者需自行承担以下责任：

- **GitHub Actions 使用条款：** 请遵守 [GitHub Actions 文档](https://docs.github.com/actions) 中的合理使用政策。过度或商业化的使用可能导致账号或服务受限。
- **代码签名与合规：** 生成的 IPA 使用伪造签名（`ldid`），不具备 Apple 官方签名。不得用于上架 App Store 或商业分发。如需正式发布，请使用有效的 Apple 开发者证书。
- **风险自负：** 本项目不对构建结果的安全性、合法性或可用性做任何保证。使用本项目所造成的任何后果由使用者自行承担。

---

## 参考

- 原仓库: [StArraySharp/iOSBuilder](https://github.com/StArraySharp/iOSBuilder)
- 上游: [StArrayJaN/iOSBuilder](https://github.com/StArrayJaN/iOSBuilder)
