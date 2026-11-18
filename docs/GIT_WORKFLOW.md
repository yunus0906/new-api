# Git 分支与提交手册

本仓库是 [QuantumNous/new-api](https://github.com/QuantumNous/new-api) 的个人 fork，分支模型固定为三层：**官方镜像 → 集成开发 → 部署**。按本手册操作，任何时候 `main` 都等于官方代码，`develop` 是唯一能提交自己代码的长期分支，`release` 是服务器真正拉取的分支。

---

## 0. 一屏速查

| 我想做什么 | 命令 |
| --- | --- |
| 同步官方更新 | `git switch main && git pull --ff-only && git switch develop && git merge main && git push` |
| 开新功能 | `git switch -c feature/xxx develop` |
| 合并功能 | `git switch develop && git merge --no-ff feature/xxx && git push` |
| 发布部署 | `git switch release && git merge --ff-only develop && git push` |
| 看当前状态 | `git status -sb && git log --oneline --graph -10` |
| 撤销还没推的提交 | `git reset --soft HEAD~1` |
| 撤销已推的提交 | `git revert <sha>` |

---

## 1. 分支模型

```
QuantumNous/new-api  (官方仓库 upstream)
          │  fetch / 只读
          ▼
       main          官方镜像：只允许 fast-forward，永不提交
          │  merge
          ▼
     develop         集成分支：你所有代码在这里，测试也在这里
          │  merge --ff-only
          ▼
     release         部署分支：服务器只拉这个
          ▲
          │  merge （功能做完即删）
     feature/xxx      一次性分支：一个功能一条命
```

| 分支 | 来源 | 用途 | 允许谁合进来 | 是否 push 到 origin |
| --- | --- | --- | --- | --- |
| `main` | `upstream/main` | 官方镜像，唯一作用是"官方更新入口" | 只允许 `upstream/main` 快进 | 是（只镜像） |
| `develop` | 由 `main` 合并而来 | 日常开发、联调、自测 | `feature/*`、`main` | 是 |
| `release` | 由 `develop` 合并而来 | 部署，服务器拉这个分支 | 只允许 `develop` 快进 | 是 |
| `feature/*` | 由 `develop` 拉出 | 单个功能/修复 | 无 | 可选（默认不推） |

---

## 2. 五条铁律

1. **`main` 上永远不写自己的代码。** 它只能是官方代码的复印件。`pre-commit` 钩子会直接拦住你。
2. **`feature/*` 只能从 `develop` 拉出，也只能合回 `develop`。** 不跨层合并。
3. **所有改动最终都进 `develop`。** `develop` 是一切的汇合点，也是唯一允许解决冲突的地方。
4. **`release` 只接受来自 `develop` 的 fast-forward。** 不允许在 `release` 上直接提交，也不允许 `--no-ff` 掩盖分叉。
5. **官方更新只能自上而下流动**：`upstream/main → main → develop → release`。反过来（把 develop 合进 main）就是上次乱掉的根源。

---

## 3. 一次性环境配置

### 3.1 远程

```bash
git remote -v
# origin    https://github.com/yunus0906/new-api.git   ← 你的 fork，可写
# upstream  https://github.com/QuantumNous/new-api.git ← 官方，只读

# 若没有 upstream：
git remote add upstream https://github.com/QuantumNous/new-api.git
git config --local remote.upstream.tagOpt --no-tags   # 不跟着拉官方 tag
```

### 3.2 Git 配置

```bash
git config --local branch.main.mergeOptions --ff-only   # main 上 pull 只能快进
git config --local push.default simple                  # 只推当前分支，不误推一堆
git config --local pull.ff only                         # 拒绝意外产生 merge 提交
```

> 唯一例外：在 `develop` 上 `git pull` 遇到分叉时会因 `pull.ff=only` 失败。这是故意的，此时应该显式 `git merge`，看清在合什么。

### 3.3 防误提交钩子（本地，只对你生效）

钩子放在 `.git/hooks/`，不进入仓库。**必须是 LF 换行**，否则会报 `bad interpreter` 而静默失效。

`.git/hooks/pre-commit` 和 `.git/hooks/pre-merge-commit`（两份内容相同，因为 merge 提交只触发后者）：

```sh
#!/bin/sh
branch=$(git symbolic-ref --short -q HEAD) || exit 0
[ "$branch" = "main" ] || exit 0
echo "blocked: main 只镜像 upstream，请 git switch develop 后提交" >&2
exit 1
```

`.git/hooks/pre-push`（放行"本地 main 恰等于 upstream/main 且是快进"的镜像推送，其余推 main 一律拒绝）：

```sh
#!/bin/sh
zero=0000000000000000000000000000000000000000
up=$(git rev-parse --verify -q refs/remotes/upstream/main) || up=""
blocked=0
while read -r lref lsha rref rsha; do
  [ "$rref" = "refs/heads/main" ] || continue
  if [ -n "$up" ] && [ "$lsha" = "$up" ] && [ "$rsha" != "$zero" ] \
     && git merge-base --is-ancestor "$rsha" "$lsha"; then
    continue
  fi
  echo "blocked: 拒绝对 main 的非镜像推送（local=$lsha remote=$rsha）" >&2
  blocked=1
done
[ "$blocked" -eq 0 ] || exit 1
```

校验换行（三个都必须是 `0`）：

```bash
LC_ALL=C grep -c $'\r' .git/hooks/pre-commit .git/hooks/pre-merge-commit .git/hooks/pre-push
```

### 3.4 换机器/重新克隆时

钩子不会跟着仓库走，需要按 3.3 重新创建。其余配置跑一遍：

```bash
git remote add upstream https://github.com/QuantumNous/new-api.git
git config --local branch.main.mergeOptions --ff-only
git config --local push.default simple
git config --local remote.upstream.tagOpt --no-tags
```

---

## 4. 日常场景

### 4.1 同步官方更新（最常用）

```bash
# 1) 拉官方代码
git fetch upstream --prune

# 2) main 快进到官方最新（只允许 --ff-only）
git switch main
git merge --ff-only upstream/main
git push origin main

# 3) 合进 develop（冲突只在这里解）
git switch develop
git merge main
git push origin develop

# 4) 自测通过后进 release
git switch release
git merge --ff-only develop
git push origin release

# 5) 服务器
git pull origin release
```

把 2～4 步合成一条（推荐日常用这个）：

```bash
git switch main && git merge --ff-only upstream/main && git push origin main \
  && git switch develop && git merge main && git push origin develop
```

**验证**：

```bash
git rev-list --left-right --count main...upstream/main   # 期望 0 0
git rev-list --left-right --count upstream/main...develop # 期望 0 N（只多你自己的提交）
```

### 4.2 开发一个新功能

```bash
git switch develop
git pull --ff-only
git switch -c feature/payment-rsa      # 命名：feature/短横线描述

# ……写代码、本地跑通、分批提交（见第 5 节）……

git push -u origin feature/payment-rsa  # 可选：想在别的机器接着写才推
```

一个功能只做一件事。做完了先自测，再合并。

### 4.3 功能完成，合并回 develop

```bash
# 先把 develop 上别人的改动合进来，冲突在自己的分支里解决
git switch feature/payment-rsa
git merge develop

# 再进 develop，保留一条可读的功能线
git switch develop
git merge --no-ff feature/payment-rsa -m "feat(payment): 易支付支持 RSA 签名"
git push

# 删掉用完的分支
git branch -d feature/payment-rsa
git push origin --delete feature/payment-rsa   # 若推过远程
```

`--no-ff` 的作用：合并后会留下一个 merge 提交，`git log --graph` 里能一眼看出"这一坨是一个功能"，回滚时 `git revert -m 1 <merge-sha>` 一次撤销整个功能。

### 4.4 部署

服务器只拉 `release`：

```bash
git fetch origin
git reset --hard origin/release     # 服务器上不要有本地改动
docker compose up -d --build
```

部署前在本地确认：

```bash
git rev-list --left-right --count release...develop   # 必须是 0 0，否则 release 不是最新
```

### 4.5 回滚

```bash
# 撤销某个已发布的提交（安全，保留历史）
git switch release
git revert <sha>
git push

# 撤销整个功能（merge 提交）
git revert -m 1 <merge-sha>

# 撤销本地还没推的提交
git reset --soft HEAD~1     # 保留改动在暂存区
git reset --hard HEAD~1     # 连改动一起丢

# 只是临时切到旧版本看看
git switch --detach <sha>
```

**已推到共享分支的提交，一律用 `revert`，不要 `reset --hard` + 强推。**

---

## 5. 提交信息规范

格式：

```
<type>(<scope>): <一句话说清做了什么>

<为什么这么做，可选>
```

| type | 用途 |
| --- | --- |
| `feat` | 新功能 |
| `fix` | 修 bug |
| `perf` | 性能优化 |
| `refactor` | 重构，行为不变 |
| `docs` | 文档 |
| `chore` | 依赖、构建、配置、杂务 |
| `test` | 测试 |
| `style` | 格式，不影响逻辑 |

scope 用模块名：`relay` / `web` / `model` / `channel` / `billing` / `i18n` / `auth` / `git`。

示例：

```
feat(payment): 易支付支持 RSA 签名
fix(relay): 请求体超过 10MB 时被截断
chore(deps): 升级 moejs 到 v0.1.0-alpha.3
docs(git): 补充分支模型与急救指令
```

提交纪律：

- **一次提交只做一件事。** 混了格式化改动的提交，review 和回滚都会很痛苦。
- 不要提交 `data/`、`.env`、`logs/`、`node_modules`、编译产物（`.gitignore` 已覆盖）。
- 提交前跑 `git status` 和 `git diff --staged`，确认没有顺手带进去的调试代码。

---

## 6. 冲突处理

### 6.1 原则

- 冲突**只在 `develop`（或你自己的 `feature/*`）上解决**，永远不要在 `main` 和 `release` 上解决。
- 解决完必须能编译、能跑起来，再 `git commit`。

### 6.2 通用流程

```bash
git merge main                     # 报冲突
git status                         # 看 both modified 的文件
# 手工编辑，删掉 <<<<<<< ======= >>>>>>> 标记
git add <冲突文件>
git commit                         # 完成合并
```

放弃这次合并：

```bash
git merge --abort
```

### 6.3 `docker-compose.yml` 的固定处理

这个文件被本地大改过（换成了自己的 mysql + 1panel 网络 + 真实连接串），官方每次动它都会冲突。**保留本地版本**：

```bash
git checkout --ours -- docker-compose.yml     # ours = 当前分支 = develop = 你的版本
git add -- docker-compose.yml
git commit --no-edit
```

方向记牢：`git switch develop` 后 `git merge main`，**`--ours` 是 develop（你的），`--theirs` 是官方**。用反了会把 MySQL 连接串、网络名全部静默换回官方默认值，`docker compose up` 直接连错库。

代价：官方对该文件的新增配置会整批丢掉。想根治，把个性化搬到 `docker-compose.override.yml`（加入 `.gitignore`），主文件对齐官方。

---

## 7. 急救手册

| 症状 | 处理 |
| --- | --- |
| 不小心在 main 上提交了 | `git switch develop && git cherry-pick <sha>` 保住代码，然后 `git switch main && git reset --hard upstream/main` |
| 已经把 dirty main 推到 origin 了 | 确认 develop 里有这些提交，再 `git push --force-with-lease=refs/heads/main:<推之前的远端 sha> origin main` |
| merge 到一半想放弃 | `git merge --abort` |
| rebase 到一半想放弃 | `git rebase --abort` |
| 想撤销还没推的 merge | `git reset --hard ORIG_HEAD` |
| 强推/误 reset 后想找回提交 | `git reflog`，找到 sha 后 `git switch -c rescue <sha>` |
| 想找回误删的分支 | `git reflog` 里搜分支 tip，`git branch <name> <sha>` |
| 提交信息写错了（还没推） | `git commit --amend` |
| 忘了切分支就改了一堆代码 | `git stash` → 切分支 → `git stash pop` |
| 提交里混进了不该提交的文件 | `git rm --cached <file>` 并加进 `.gitignore` |
| `pull.ff=only` 导致的 pull 失败 | 说明本地和远端分叉了，改用 `git fetch` + `git log --graph` 看清再 `git merge` |

---

## 8. 提交/推送前检查清单

**提交前**

- [ ] 在 `develop` 或 `feature/*` 上（`git branch --show-current`）
- [ ] `git status` 里没有 `data/`、`.env`、日志、编译产物
- [ ] `git diff --staged` 看过了
- [ ] 提交信息符合第 5 节格式

**推送前**

- [ ] `git fetch origin && git status -sb` 确认没有被别人更新的分支
- [ ] 推的是当前分支：`git push origin HEAD`
- [ ] 绝不对 `main`、`release` 使用 `--force`

**部署前**

- [ ] `git rev-list --left-right --count release...develop` = `0 0`
- [ ] `develop` 上自测通过
- [ ] 有回滚方案（记住这次要 revert 的 sha）

---

## 9. 为什么会"乱"，以及怎么不再乱

分支乱掉的典型路径：

1. 在 `main` 上直接改代码并提交 → main 不再等于官方
2. 把 `develop` 合进 `main`（方向反了）→ main 永久带上自己的代码
3. 用 `git pull` 在 `main` 上产生 merge 提交 → 官方更新和自己的代码缠在一起
4. 同一个文件同时出现在 `main`、`develop`、`release` 三条分支的多条提交里 → 冲突越解越乱

对应的防呆措施就是第 2 节那五条铁律，加上第 3.3 节的钩子。**只要 `main` 永远等于 `upstream/main`，这个仓库就不会再乱。**

---

相关文档：[本地开发启动指南](./START_DEV.md)（环境、构建、调试）。
