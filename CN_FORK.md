# 国服（CN）支持分支：自动跟进上游 + 自动打补丁 + 自动出 APK

这个 fork **不保存修改过的源码**，只保存一个补丁和一条工作流。每次构建时：

```
upstream/main  ──merge──▶ 本仓库 main ──apply patch──▶ 编译 ──▶ Release（cn-latest）
                 (每天自动)                (git apply -3)
```

- 补丁：`patches/0001-add-china-cn-region-support.patch`（`git format-patch` 生成，可直接 upstream PR）
- 工作流：`.github/workflows/cn-build.yml`（每天北京时间 11:00 + 手动触发）
- 产出：Releases 里的滚动发布 `cn-latest` → `WoW-Forever-CN-<版本>-<上游sha>.apk`（`app.wowforever.cn`，可与国际版共存）

## 行为

每次运行会：

1. `git fetch upstream main`，如果上游有更新就 `git merge` 进本仓库 `main` 并 push（合并冲突则 job 失败并在日志里指出冲突文件）
2. 把补丁以三方合并方式打到上游源码上（`git apply -3`）；**补丁打不上也会直接失败**，不会产出半残的包
3. 编译浏览器登录用的 `wow-session-bridge.exe`（LLVM-MinGW，源码在 `tools/wow-session-bridge/`，不入库）
4. 跑 410 个单元测试 → `assembleModernDebug`
5. 用 `CN_*` secrets 里的签名（本机原有的 debug 签名，保证能覆盖安装）发布到 `cn-latest`

## 需要配置的 Secrets（已配置）

| Secret | 说明 |
| :--- | :--- |
| `CN_KEYSTORE_BASE64` | 签名库（本机 `~/.android/debug.keystore` 的 base64，保证签名不变、可直接覆盖安装） |
| `CN_STORE_PASSWORD` | 库口令 |
| `CN_KEY_ALIAS` | key 别名 |
| `CN_KEY_PASSWORD` | key 口令 |

没配置时回退到 CI 临时 debug 签名（**每次签名都会变，无法覆盖安装**，只能全新安装）。

## 上游更新后自动跟进

- 定时任务每天拉一次上游；上游发新版后，下一次构建就带上新代码（补丁若与上游改动冲突，构建会失败，见下面的修法）
- 想立刻构建：Actions → **CN build (国服补丁)** → Run workflow

## 补丁冲突了怎么办

```bash
git clone https://github.com/<you>/wow-forever-android && cd wow-forever-android
git remote add upstream https://github.com/jaredgei/wow-forever-android
git fetch upstream && git checkout -b cn upstream/main
git apply -3 --verbose patches/0001-add-china-cn-region-support.patch   # 冲突文件会标出来
# 手工解决冲突 → 重新生成补丁 → 提交 patches/ 并 push
git add -A && git commit -m "update CN patch"
git format-patch -1 --stdout > patches/0001-add-china-cn-region-support.patch
```

## 回上游贡献

补丁按单 commit 组织，可以直接：

```bash
gh pr create --repo jaredgei/wow-forever-android \
  --head <you>:cn-edition --base main \
  --title "Add China (CN) region support" \
  --body-file <(sed -n '/^Adds the 国服 region/,$p' patches/0001-add-china-cn-region-support.patch | head -40)
```

（或在本仓库 `main` 上 `git format-patch` 的提交信息里直接取描述。）

## 国服相关的事实记录（供上游 review）

- 国服版本表：`https://cn.version.battlenet.com.cn/v2/products/wowf`，`cn` 行的 BuildConfig/CDNConfig 与美服完全相同（各区共用同一个 Windows ARM64 客户端），CDN 指向 `blzdist-wow.necdn.leihuo.netease.com`
- 国服安装清单里有 `Wow-ARM64.exe`（Windows + arm64 标签），并有 `zhCN` 文本/语音标签
- 国服登录服务器是 **`cn.actual.battlenet.com.cn:1119`**（不是 `cn.actual.battle.net`）
- 国服已切到网易账号 SSO：`Connection.log` 显示密码直连在 `credentials submission` 阶段被服务端 `ERROR_NETWORK_MODULE_SOCKET_CLOSED (1016)` 断开，因此国服默认走浏览器登录
- `Config.wtf`：`portal "CN"`、`agentUID "wowf"`、`textLocale/audioLocale "zhCN"`