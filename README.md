# iOSBuilder

自动化 iOS IPA 构建工作流，基于 GitHub Actions，无需本地 Mac 即可打包。

## 使用方式

### 1. Fork 本仓库

点击右上角 **Fork** 按钮，将仓库复制到你的 GitHub 账号下。

### 2. 替换你的 Unity 导出工程

编辑 `download.sh`（或 `download2.sh`），将下载链接换成你自己的 Unity 导出工程压缩包：

```bash
# 修改前
curl -LO https://github.com/StArrayJaN/iOSBuilder/releases/download/1.0/ADOFAI292.zip

# 修改后 — 换成你的下载地址
curl -LO https://github.com/你的用户名/iOSBuilder/releases/download/1.0/YourProject.zip
```

**你的压缩包需要包含：**
- `Unity-iPhone.xcodeproj`（或 `.xcworkspace`）
- 所有源码和资源文件
- （可选）`Info.plist` — 如需自定义

### 3. 上传压缩包到 Releases

```bash
# 在本地打包
zip -r YourProject.zip Unity-iPhone.xcodeproj/ Assets/ ...

# 在 GitHub 仓库页面 → Releases → 创建新 Release
# 上传 YourProject.zip 作为附件
```

### 4. 触发构建

- 进入你的 Fork 仓库 → **Actions** → **Build IPA directly**
- 点击 **Run workflow** → 选择分支 → 绿色按钮
- 等待 10-20 分钟构建完成
- 在运行记录页面下载生成的 `.ipa` Artifact

### 5. 进阶配置

编辑 `.github/workflows/build.yml` 中的环境变量：

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `scheme` | Xcode Scheme 名称 | `Unity-iPhone` |

## 注意事项

- 构建环境为 `macos-latest`，使用 Xcode 26.2
- 使用 `ldid` 做 fake-sign，生成的 IPA **不包含有效开发者签名**
- 该 IPA 仅可在模拟器或越狱设备上安装使用
- 如需真机安装，需自行替换代码签名证书
