# CocoaPods 私有 Specs 推送问题排查与修复指南

本文整理了在使用 CocoaPods 向私有 Specs 仓库执行 `pod repo push` 时的常见错误与修复步骤。内容已做脱敏处理，不包含任何项目名称、版本号或私有仓库地址。

适用读者：维护私有 CocoaPods Specs 仓库的 iOS 工程师。

---

## 一、典型报错与现象

- `[!] The repo '<PRIVATE_SPECS_REPO>' is not clean`
- `fatal: cannot create directory at '...': Permission denied`
- 工作区出现未跟踪（`??`）文件/目录，导致仓库处于“脏”状态

---

## 二、根因分析

1) 工作区不干净：
   - 手动拷贝、临时生成的文件/目录未被 Git 跟踪；
   - 之前的失败操作生成了未跟踪目录，阻塞后续 `pod repo push`。

2) 目录权限异常：
   - 部分目录由 `root` 拥有（常见于曾用 `sudo` 执行过 `pod` 或 Git 操作），普通用户无法在该目录下创建版本文件，`git pull` 报 `Permission denied`。

3) 其他杂项：
   - `.DS_Store` 等系统文件未忽略，频繁造成“not clean”。

---

## 三、快速修复步骤（推荐）

> 以下命令均以私有 Specs 仓库路径 `~/.cocoapods/repos/<PRIVATE_SPECS_REPO>` 为例，请替换为你的仓库名称。

1) 检查工作区状态（确认“脏点”）：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> status --porcelain
```

2) 临时清理未跟踪/未提交文件（安全可回滚）：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> stash -u
```

3) 修复目录所有权（如遇 `Permission denied`）：

```
sudo chown -R "$USER":staff ~/.cocoapods/repos/<PRIVATE_SPECS_REPO>
find ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> ! -user "$USER" -maxdepth 2 -ls
```

4) 更新私有 Specs 仓库：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> pull
```

5) 执行推送（示例）：

```
env isDevelop=1 \
pod repo push <PRIVATE_SPECS_REPO> <PODSPEC_FILE>.podspec \
  --private --verbose --allow-warnings \
  --skip-import-validation --use-libraries \
  --sources=<PRIVATE_SPECS_GIT_URL>
```

> 说明：
> - `<PODSPEC_FILE>.podspec` 为你要推送的 podspec 文件名；
> - `<PRIVATE_SPECS_GIT_URL>` 为你的私有 Specs 仓库地址；
> - 如非 CI/root 环境，避免使用 `--allow-root` 与 `sudo`。

6) 验证推送：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> log -1 --oneline
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> ls-remote origin -h refs/heads/master
```

7) 清理临时 stash（可选）：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> stash list
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> stash drop stash@{0}
# 视情况依次清理 stash@{1}、stash@{2} ...
```

---

## 四、彻底清理方案（会丢弃本地修改）

若确定本地改动无需保留，可直接恢复到远端版本：

```
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> reset --hard HEAD
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> clean -fd
git -C ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> pull
```

如仓库损坏或远端地址更换，可重新添加：

```
pod repo remove <PRIVATE_SPECS_REPO>
pod repo add <PRIVATE_SPECS_REPO> <PRIVATE_SPECS_GIT_URL>
```

---

## 五、预防复发的最佳实践

- 避免使用 `sudo` 或以 root 身份执行 `pod`/Git 操作；
- 如需 CI 以 root 运行，限制在 CI 环境中使用 `--allow-root`，本地不要使用；
- 设置全局忽略以过滤系统文件：

```
git config --global core.excludesfile ~/.gitignore_global
echo .DS_Store >> ~/.gitignore_global
```

- 定期巡检并修复所有权（如遇异常）：

```
find ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> ! -user "$USER" -maxdepth 2 -ls
sudo chown -R "$USER":staff ~/.cocoapods/repos/<PRIVATE_SPECS_REPO>
```

---

## 六、问题对照速查

- 报错：`[!] The repo '<...>' is not clean`
  - 处理：`git status --porcelain` 定位未跟踪/未提交文件 → `git stash -u` 清理 → 重试。

- 报错：`fatal: cannot create directory at '...': Permission denied`
  - 处理：目录被 root 拥有 → `sudo chown -R "$USER":staff ~/.cocoapods/repos/<PRIVATE_SPECS_REPO>` → `git pull` → 重试。

- 反复出现 `.DS_Store` 导致不干净
  - 处理：添加全局忽略 `.DS_Store`，并删除历史遗留：

```
find ~/.cocoapods/repos/<PRIVATE_SPECS_REPO> -name .DS_Store -delete
```

---

如需用于团队外部分享，可直接分发本文档；命令中的占位符需替换为你自己的仓库名与地址。

