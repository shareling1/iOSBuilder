# iOSBuilder

An automated iOS IPA build workflow powered by GitHub Actions. No local Mac required — it automatically compiles, fake-signs, and packages IPA from any engine's Xcode export.

---

## How It Works

```
Engine Xcode Project.zip → GitHub Releases
         ↓
GitHub Actions spins up macOS runner
         ↓
download.sh downloads zip → extracts → Xcode project ready
         ↓
xcodebuild clean archive → compiles .app
         ↓
ldid fake-signs (no Apple dev cert needed)
         ↓
Packaged into .ipa → uploaded as Actions Artifact
         ↓
Download .ipa from Actions page
```

---

## Switching Engines

This workflow compiles **Xcode projects** — it doesn't care which engine produced them. As long as your engine can export an iOS Xcode project (`.xcodeproj` / `.xcworkspace`), it works.

| Engine | How to Export | What to Change |
|--------|---------------|----------------|
| **Unity** (current) | File → Build Settings → iOS → Export | Nothing extra |
| **Unreal** | Platforms → iOS → Package Project | Set `scheme` to project name; output is usually `.xcworkspace` |
| **Godot** | Export → iOS → Export | Set `scheme` to project name |
| **Cocos2d-x** | Ships with Xcode project | Set `scheme` to project name |
| **Native Xcode** | Create directly in Xcode | Set `scheme` to target name |

**What needs changing:**
1. `build.yml` → `scheme` env var — set to your Xcode project's Scheme name
2. `download.sh` → download URLs — point to your engine's exported zip
3. Zip contents — make sure `.xcodeproj` or `.xcworkspace` is at the root

The workflow auto-detects `.xcworkspace` vs `.xcodeproj` — no manual config needed.

---

## Quick Start

### 1. Prepare Your Xcode Project

Export the iOS Xcode project from your engine and zip it up.

**Unity example:**
- **Il2Cpp backend:** Export MainProject + Il2CppOutputProject separately → use `download.sh`
- **Mono backend:** One complete zip → use `download2.sh`

### 2. Upload Zip to Releases

Go to your GitHub repo → **Releases** → **Create a new release** → upload zip → publish.

### 3. Modify the Download Script

Update the download URLs to point to your Release.

If using `download.sh`:

```bash
# Before
curl -LO https://github.com/StArrayJaN/iOSBuilder/releases/download/1.0/MainProject.zip
curl -LO https://github.com/StArrayJaN/iOSBuilder/releases/download/1.0/Il2CppOutputProject.zip

# After
curl -LO https://github.com/your-username/your-repo/releases/download/0.1/YourProject.zip
```

If using `download2.sh`:

```bash
# Replace version number with sed
sed -i "s|1.0|0.1|g" download2.sh

# Or replace the entire download line
sed -i "5d" download2.sh
sed -i "5icurl -LO https://github.com/your-username/your-repo/releases/download/0.1/XcodeProject.zip" download2.sh
```

### 4. Trigger a Build

| Method | Action |
|--------|--------|
| Manual | **Actions** → **Build IPA directly** → **Run workflow** |
| On Push | Auto-runs on `git push` |
| On Release | Auto-runs when creating or editing a Release |

### 5. Download IPA

After the build finishes, go to the workflow run page and download the `.ipa` under **Artifacts**.

---

## File Reference

### `.github/workflows/build.yml`

The core CI pipeline — 10 steps:

| Step | Purpose |
|------|---------|
| setup-xcode | Install specified Xcode version |
| Checkout | Pull your repo |
| Run download script | Execute `download.sh` or `download2.sh` |
| Set Scheme | Auto-detect or use configured Scheme |
| Set filetype_parameter | Detect `.xcworkspace` vs `.xcodeproj` |
| Set file_to_build | Locate the project file |
| Build and Archive | `xcodebuild clean archive` |
| Install ldid | Install fake-signing tool |
| Fakesign | Fake-sign all frameworks and the .app |
| Create IPA | Package `.app` into `.ipa` and upload |

### `download.sh` — Il2Cpp version

For Unity + Il2Cpp backend:

- Downloads `MainProject.zip` (main project)
- Downloads `Il2CppOutputProject.zip` (Il2Cpp output)
- Downloads `Info.plist` (overrides default)
- Sets permissions: `usymtool`, `usymtoolarm64`

### `download2.sh` — Mono version

For Unity Mono backend. Requires only a single `XcodeProject.zip`.

---

## Syncing Upstream Changes

If your repo is forked from [StArraySharp/iOSBuilder](https://github.com/StArraySharp/iOSBuilder) or [StArrayJaN/iOSBuilder](https://github.com/StArrayJaN/iOSBuilder):

```bash
# Add upstream remote
git remote add upstream https://github.com/StArrayJaN/iOSBuilder.git

# Pull and merge (allow unrelated histories)
git pull upstream main --allow-unrelated-histories

# Resolve conflicts if any, then commit
git add .
git commit -m "Merge upstream changes"
git push
```

---

## Cleaning Large Files from Git History

If you accidentally committed a large file (e.g. `.zip`) into Git history, use `git filter-repo`:

```bash
# Install
pip install git-filter-repo

# Run in your repo (needs a fresh clone)
# Remove all traces of a specific path
git filter-repo --path '.github/workflows/XcodeProject.zip' --invert-paths --force

# Force push to overwrite remote history
git push origin --force --all
git push origin --force --tags
```

> **Note:** `git filter-repo` requires a fresh clone. If you get `Aborting: Refusing to destructively overwrite repo history`, add `--force`.

---

## Quick Push Script

If your network is unstable and `git push` often fails, create a retry script:

```bash
#!/bin/bash
echo "# minor change $(date)" >> download2.sh
git add download2.sh
git commit -m "Add comment to download2.sh"
until git push; do
    echo "Push failed, retrying in 200ms..."
    sleep 0.2
done
```

Save as `push.sh` and run `bash push.sh` whenever you need to push.

---

## FAQ

### Q: Can the IPA be installed on an iPhone?

**Not directly.** The IPA is fake-signed with `ldid` and has no Apple Developer certificate.

| Use Case | Solution |
|----------|----------|
| Jailbroken device / TrollStore | Works as-is |
| Dev device debugging | Re-sign with your dev cert using `codesign` |
| App Store submission | Requires Xcode + paid account — not covered here |

### Q: Build failed — what now?

Click the failed run in Actions and check the logs. Common issues:

| Error | Cause | Fix |
|-------|-------|-----|
| `.app` not found | Wrong Scheme name | Check `scheme` in `build.yml` |
| `Unity-iPhone.xcodeproj` not found | Wrong zip structure | Make sure `.xcodeproj` is at zip root |
| `curl: (22) ... 404` | Bad download URL | Check Release link and version |
| `xcodebuild error` | Broken project | Verify engine export is complete |
| `filter-repo Aborting` | Not a fresh clone | Add `--force` flag |

### Q: Which script should I use — download.sh or download2.sh?

| | Mono | Il2Cpp |
|--|------|--------|
| Script | `download2.sh` | `download.sh` |
| Zip | 1 complete zip | Main project + Il2Cpp output separate |
| Unity setting | Scripting Backend → Mono | Scripting Backend → IL2CPP |

---

## References

- Original repo: [StArraySharp/iOSBuilder](https://github.com/StArraySharp/iOSBuilder)
- Upstream: [StArrayJaN/iOSBuilder](https://github.com/StArrayJaN/iOSBuilder)
