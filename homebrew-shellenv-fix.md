# Homebrew shellenv 权限问题修复记录

## 背景

- 在 macOS 14 上执行 `brew shellenv` 时出现错误：`/opt/homebrew/Library/Homebrew/cmd/shellenv.sh: line 18: /bin/ps: Operation not permitted`。
- 根因是受限环境下无法调用 `/bin/ps`，导致 Homebrew 无法获取父进程 shell 名称。

## 修改说明

- 文件：`/opt/homebrew/Library/Homebrew/cmd/shellenv.sh`
- 变更点：对 `homebrew-shellenv` 函数的 shell 判定逻辑增加容错处理。若 `/bin/ps` 调用失败，则回退为 `${SHELL##*/}`，避免抛出权限错误。

```bash
if [[ -n "$1" ]]
then
  HOMEBREW_SHELL_NAME="$1"
else
  if HOMEBREW_PS_OUTPUT="$(/bin/ps -p "${PPID}" -c -o comm= 2>/dev/null)"
  then
    HOMEBREW_SHELL_NAME="${HOMEBREW_PS_OUTPUT}"
  elif [[ -n "${SHELL}" ]]
  then
    HOMEBREW_SHELL_NAME="${SHELL##*/}"
  else
    HOMEBREW_SHELL_NAME=""
  fi
fi
```

## 验证步骤

1. 启动新的 zsh 会话（或确保未手动指定 shell 名称）。
2. 执行 `brew shellenv`。
3. 期望输出仅包含环境变量导出语句，不再出现 “Operation not permitted” 警告。

示例输出：

```bash
export HOMEBREW_PREFIX="/opt/homebrew";
export HOMEBREW_CELLAR="/opt/homebrew/Cellar";
export HOMEBREW_REPOSITORY="/opt/homebrew";
fpath[1,0]="/opt/homebrew/share/zsh/site-functions";
eval "$(/usr/bin/env PATH_HELPER_ROOT="/opt/homebrew" /usr/libexec/path_helper -s)"
[ -z "${MANPATH-}" ] || export MANPATH=":${MANPATH#:}";
export INFOPATH="/opt/homebrew/share/info:${INFOPATH:-}";
```

若输出如上且无报错，说明修复生效。
