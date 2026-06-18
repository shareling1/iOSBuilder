# iOSBuilder

自动化 iOS IPA 构建工作流，基于 GitHub Actions 实现 **无 Mac 电脑也能打包 iOS 应用**。

支持从 Unity 导出的 Xcode 工程自动完成：下载资源 → 编译打包 → 伪造签名 → 生成 IPA → 上传构件，全流程无人值守。

---

## 目录

- [工作原理](#工作原理)
- [快速开始](#快速开始)
- [触发构建的三种方式](#触发构建的三种方式)
- [文件说明](#文件说明)
- [工作流各步骤详解](#工作流各步骤详解)
- [自定义配置](#自定义配置)
- [常见问题](#常见问题)

---

## 工作原理

整个流程在你的 GitHub 仓库的 Actions 虚拟机（`macos-latest`）上运行，不需要你自己的电脑：

```
你上传 Unity 导出工程.zip → GitHub Releases
         ↓
GitHub Actions 启动 macOS 虚拟机
         ↓
download.sh 下载 zip → 解压 → 得到 Xcode 工程
         ↓
xcodebuild clean archive → 编译出 .app
         ↓
ldid 伪造签名（免开发者证书）
         ↓
打包成 .ipa → 上传到 Actions Artifact
         ↓
你去 Actions 页面下载 .ipa
```

---

## 快速开始

### 1. Fork 本仓库

点击 GitHub 页面右上角的 **Fork**，把仓库复制到你自己的账号下。

### 2. 准备 Unity 导出的 Xcode 工程

在 Unity 中导出 iOS Xcode 工程后，将所有文件打包为一个 zip：

```bash
# 假设 Unity 导出的目录名为 Unity-iPhone/
zip -r XcodeProject.zip Unity-iPhone/
```

> 压缩包根目录下必须包含 `Unity-iPhone.xcodeproj` 或 `.xcworkspace`，工作流会自动识别。

如果你的 Unity 工程使用了 **Il2Cpp** 后端，需要额外提供 Il2Cpp 输出文件（参考 `download.sh` 的做法）。

### 3. 上传压缩包到 Releases

1. 进入你的 Fork 仓库 → 右侧 **Releases** → **Create a new release**
2. Tag version 填 `1.0`（或任意你喜欢的版本号）
3. 上传 `XcodeProject.zip`
4. 点击 **Publish release**

### 4. 修改 download 脚本

根据你的情况，编辑 `download.sh` 或 `download2.sh`：

```bash
# 修改 download 脚本中的下载链接
# 把原来的链接换成你自己 Release 的下载地址
curl -LO https://github.com/你的用户名/iOSBuilder/releases/download/1.0/XcodeProject.zip
```

然后在 `.github/workflows/build.yml` 中确认使用的是哪个脚本：

```yaml
# 第 29-31 行，默认使用 download.sh
- name: Run download script
  run: |
    chmod +x ./download.sh
    ./download.sh

# 如果你想用 download2.sh，改成：
#  ./download2.sh
```

### 5. 触发构建

参见下方 [触发构建的三种方式](#触发构建的三种方式)。

### 6. 下载 IPA

构建完成后：
1. 进入仓库 **Actions** 页面
2. 点击成功运行的那个工作流
3. 在 **Artifacts** 区域点击 `.ipa` 文件下载

---

## 触发构建的三种方式

### 方式 A：手动触发（推荐调试时用）

1. 仓库页面 → **Actions** → **Build IPA directly**
2. 点击 **Run workflow** 下拉按钮
3. 选择分支（默认 `main`），点绿色 **Run workflow**
4. 等待 15-25 分钟运行完成

### 方式 B：推送代码自动触发

当你向仓库推送代码（`git push`）时，工作流会自动运行。适合每次修改脚本后自动验证。

### 方式 C：发布 Release 自动触发

当你创建或编辑 Release 时，工作流会自动运行。适合正式发版场景。

---

## 文件说明

### `.github/workflows/build.yml` — 核心 CI 流程

完整定义了 10 个步骤：

| 步骤 | 作用 |
|------|------|
| setup-xcode | 安装 Xcode 26.2（macOS 环境自带，此步骤确保版本正确） |
| Checkout | 拉取你的仓库代码 |
| Run download script | 执行 `download.sh`，下载 Unity 导出工程 |
| Set Scheme | 自动检测或使用预设的 Xcode Scheme |
| Set filetype_parameter | 自动识别是 `.xcworkspace` 还是 `.xcodeproj` |
| Set file_to_build | 找到需要编译的工程文件 |
| Build and Archive | 执行 `xcodebuild clean archive` 编译 |
| Install ldid | 安装伪造签名工具 |
| Fakesign | 对所有 Framework 和应用本身执行伪造签名 |
| Create IPA | 将 `.app` 放入 Payload 目录并压缩为 `.ipa` |
| Upload IPA | 将 `.ipa` 上传为 Actions Artifact |

### `download.sh` — 下载脚本（复杂版）

适用于 Unity + Il2Cpp 导出：

| 操作 | 说明 |
|------|------|
| 下载 `ADOFAI292.zip` | 主工程（含 `Unity-iPhone.xcodeproj`） |
| 下载 `Il2CppOutputProject.zip` | Il2Cpp 输出工程 |
| 下载 `Info.plist` | 覆盖默认的 Info.plist |
| 验证 `Unity-iPhone.xcodeproj` 存在 | 确保解压正确 |
| 设置执行权限 | `usymtool`, `usymtoolarm64` |

### `download2.sh` — 下载脚本（简化版）

适用于 Unity **非 Il2Cpp**（Mono）导出，或者你已经将 Il2Cpp 合并到主包中时使用：

| 操作 | 说明 |
|------|------|
| 下载 `XcodeProject.zip` | 完整的 Xcode 工程（含所有内容） |
| 设置执行权限 | 可选地设置 `usymtool` 权限 |

> **如何选择？**
> - Unity 的 **Project Settings → Player → Configuration → Scripting Backend** 选 **IL2CPP** → 用 `download.sh`
> - 选 **Mono** → 用 `download2.sh`（只需一个 zip）

---

## 自定义配置

### 修改 Xcode Scheme

编辑 `.github/workflows/build.yml` 中的环境变量：

```yaml
env:
  scheme: Unity-iPhone          # 改为你的 Scheme 名称
  archive_path: archive
```

如果你不确定 Scheme 名称，可以设为 `default`，工作流会自动检测：

```yaml
env:
  scheme: default               # 自动选择第一个可用 scheme
```

### 修改 Xcode 版本

```yaml
- uses: maxim-lobanov/setup-xcode@v1
  with:
    xcode-version: '26.2'      # 改为你需要的版本
```

可用版本参考 [setup-xcode 文档](https://github.com/maxim-lobanov/setup-xcode)。

### 修改输出 IPA 名称

IPA 文件默认以 Scheme 名称命名。如需自定义，修改 `build.yml` 第 96 行：

```yaml
zip -r "MyApp.ipa" Payload -x "._*" -x ".DS_Store" -x "__MACOSX"
```

---

## 常见问题

### Q：生成的 IPA 能安装到 iPhone 上吗？

**不能直接安装。** 因为工作流使用 `ldid` 做伪造签名（fake-sign），没有使用 Apple 开发者证书。

**解决方案：**

| 用途 | 做法 |
|------|------|
| 越狱手机 | 直接用，也可以用 TrollStore 安装 |
| 开发者真机调试 | 下载 IPA 后，用 `codesign` 重新签名你的开发者证书 |
| App Store 上架 | 需用 Xcode + 付费开发者账号 + App Store Connect 提交流程，不适用本项目 |

### Q：构建失败怎么办？

点击 Actions 中失败的任务，查看日志。常见原因：

| 错误 | 原因 | 解决方法 |
|------|------|----------|
| `未找到 .app 文件` | 编译没生成 .app | 检查 Scheme 名称是否正确 |
| `未找到 Unity-iPhone.xcodeproj` | 压缩包结构不对 | 确保 zip 解压后根目录有 `.xcodeproj` |
| `curl: (22) ... 404` | 下载链接失效 | 检查 Release 是否发布、URL 是否正确 |
| `xcodebuild: error: ...` | 编译错误 | 检查 Unity 导出的工程是否完整 |
| `The request timed out` | 网络超时 | 重新 Run workflow |

### Q：如何只编译不签名？

删除 `build.yml` 中 `Install ldid` 和 `Fakesign` 两个步骤即可。不过没有签名的 `.app` 无法打包为 `.ipa`。

### Q：构建时间太长？

- 首次构建约 15-25 分钟（含下载依赖）
- 后续相同依赖的构建会快一些（Actions 缓存）
- 如果你的工程很大，可以考虑把编译缓存加到 workflow 中

### Q：Il2Cpp 和 Mono 有什么区别？

| 特性 | Mono | Il2Cpp |
|------|------|--------|
| Unity 默认 | 旧项目默认 | 新项目推荐 |
| 脚本处理 | 编译为 .NET DLL | 转为 C++ 再编译为机器码 |
| 性能 | 一般 | 更好 |
| 包大小 | 较小 | 略大 |
| 对应脚本 | `download2.sh` | `download.sh` |

---

## 许可证

本项目基于原仓库 [StArraySharp/iOSBuilder](https://github.com/StArraySharp/iOSBuilder) 修改，仅供学习和个人使用。
