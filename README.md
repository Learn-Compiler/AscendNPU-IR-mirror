# AscendNPU-IR-mirror

[\https://gitcode.com/Ascend/AscendNPU-IR.git\](https://gitcode.com/Ascend/AscendNPU-IR.git) 的自动镜像仓库。
由 GitHub Actions 每 12 小时同步一次，**不需要任何 PAT 或 secret**。

## 仓库结构（单仓库方案）

\main              ← 镜像配置与同步管理脚本                   ← 默认分支，受保护
master            ┐
stable            │
release/v2.0.x    ├─ 上游镜像内容：17 个分支
feature/...       │   （含 release/... 这类带斜杠的分支）
dev/...           ┘
v2.0.0            ┐
v1.2.0            ├─ 上游镜像内容：10 个 tag
v1.1.0            │
...               ┘
\
**代码在 \master\ 等分支，不在 \main\。** \main\ 仅用于存放镜像配置和同步脚本。

为什么这么拆：GitHub 的 \schedule\ 触发器**只在默认分支的 workflow 文件上生效**，
而镜像会写满 \master\/\elease\/所有 tag。所以固定用 \main\ 当配置分支，镜像永远不碰它 ——
推送的 refspec 只取上游存在的 ref（上游没有 \main\），清理残留 ref 时也显式跳过 \main\。

## 用法

\\ash
# 克隆最新代码 (master 分支)
git clone --branch master https://github.com/Learn-Compiler/AscendNPU-IR-mirror.git

# 克隆稳定版 (stable 分支)
git clone --branch stable https://github.com/Learn-Compiler/AscendNPU-IR-mirror.git

# 克隆指定发布版本 (如 v2.0.0 tag)
git clone --branch v2.0.0 https://github.com/Learn-Compiler/AscendNPU-IR-mirror.git

# 查看相比上游领先或落后情况
cd AscendNPU-IR-mirror
git remote add upstream https://gitcode.com/Ascend/AscendNPU-IR.git
git fetch upstream && git log --oneline HEAD..upstream/master | head
\
## 同步机制

| 项 | 值 |
|---|---|
| 频率 | 每 12 小时（UTC \17 */12 * * *\，即北京时间 08:17 / 20:17） |
| 触发 | \schedule\ + 推送到 \main\ + 手动 \workflow_dispatch\ |
| 凭据 | 仓库自带的 \GITHUB_TOKEN\（\contents: write\），无需配置 |
| 增量 | 先比对两边 ref，一致就跳过克隆，不重复下载 |
| 重试 | 抓取失败自动重试 3 次（GitCode 偶发 TLS 断连） |

手动同步：Actions → **Sync AscendNPU-IR from GitCode** → Run workflow。

## 关键实现细节

**不用 \git clone --mirror\**：GitCode 会广播大量 \efs/tmp/*\ 临时 ref（网页端操作残留），
\clone --mirror\ 会把它们连同垃圾对象一起抓下来；而推回 GitHub 时，GitHub 只接受 \efs/heads/*\ 和 \efs/tags/*\，遇到 \efs/tmp/*\ 会直接报错拒绝整个 push。
所以用 \git init --bare\ + 显式 refspec，只跟踪 heads 和 tags。

**不用 \push --mirror\**：那会删掉镜像端所有上游没有的 ref —— 包括 \main\，workflow 跟着消失，
定时任务就失效。改成「显式 refspec 推送 + 逐个删除残留 ref 时跳过 \main\」。

**上游若新增 \main\ 分支**：会和本仓库的配置分支冲突。workflow 里有断言，遇到这种情况会
直接失败并报错，需要人工决策（把配置分支改名，或调整 \PROTECTED_BRANCH\）。

## 兜底：改在本机跑

如果 runner 拉 GitCode 超时，可在本机跑同步脚本：

\\ash
./scripts/sync-local.sh

# 装成每 12 小时自动跑的 systemd timer (可选)
mkdir -p ~/.local/bin ~/.config/systemd/user
cp scripts/sync-local.sh ~/.local/bin/ascendnpu-ir-mirror-sync
chmod +x ~/.local/bin/ascendnpu-ir-mirror-sync
cp scripts/ascendnpu-ir-mirror.{service,timer} ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now ascendnpu-ir-mirror.timer
\EOF
