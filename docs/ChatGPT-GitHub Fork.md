## 推荐的分支结构

```text
upstream/main        官方仓库
      │
      ▼
origin/main          只同步官方（永远不开发）
      │
      ├─────────────── feature/login
      ├─────────────── feature/model
      ├─────────────── feature/ui
      ├─────────────── feature/plugin
      │
      ▼
develop              自己的开发分支（合并所有 feature）
      │
      ▼
release / deploy     部署分支（生产）
```

职责分别是：

- **main**：只同步官方，不提交自己的代码。
- **feature/**：一个功能一个分支。
- **develop**：你的所有功能集合，日常测试都在这里。
- **release（或 deploy）**：真正上线部署的分支。

---

## 日常开发流程

假设官方更新了：

```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

然后把官方更新同步到开发分支：

```bash
git checkout develop
git merge main
```

解决一次冲突即可。

---

## 开发新功能

例如新增支付：

```bash
git checkout -b feature/payment develop
```

开发完成：

```bash
git checkout develop
git merge feature/payment
```

所有功能都进入 develop。

---

## 部署

上线时：

```bash
git checkout release
git merge develop
```

部署：

```text
release
```

服务器始终拉：

```bash
git pull origin release
```

而不是：

```text
main
```

这样就不会把官方未经验证的新代码直接部署到生产环境。

---

## 官方更新

例如官方今天更新了几十个 commit：

```text
upstream/main
        │
        ▼
main
        │
        ▼
develop
        │
        ▼
release
```

你只需要按这个顺序往下同步。

如果发现官方更新有问题，你可以先停留在 develop 修复，等测试通过后再合并到 release。

---

## 如果只有你一个人维护

如果项目规模不大，其实可以更简单：

```text
upstream/main
      │
      ▼
origin/main        (同步官方)

      │
      ▼
custom            (你的开发 + 部署)
```

平时：

```bash
git checkout main
git merge upstream/main

git checkout custom
git merge main
```

服务器一直部署：

```text
custom
```

这样只有两个长期分支：

- **main**：官方同步。
- **custom**：自己的版本（开发、测试、部署）。

---

### 结合你现在的场景（NewAPI）

如果目前只有你维护，我反而**更推荐两长期分支方案**：

```text
upstream/main
        │
        ▼
main            ← 永远同步官方
        │
        ▼
custom          ← 自己所有功能
```

服务器始终部署 **custom** 分支。

这样每次官方更新，你只需要：

```bash
git checkout main
git fetch upstream
git merge upstream/main

git checkout custom
git merge main
```

如果平时还有一些实验性的功能，再从 `custom` 拉出 `feature/*` 分支开发，完成后再合并回 `custom`。这种方式简单、清晰，而且对于个人维护的 Fork 项目来说，后续同步官方更新也比较省心。

---
