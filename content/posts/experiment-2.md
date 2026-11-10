---
title: "Shell 脚本编程实验：日志备份脚本"
date: 2026-10-26
draft: false
categories: ["Linux课程"]
tags: ["Shell", "脚本", "自动化", "crontab"]
---

## 实验目的

掌握 Shell 脚本的变量、位置参数、条件判断、循环与函数，能编写带参数校验与日志输出的运维脚本，并用 `crontab` 定时执行。

## 实验环境

Ubuntu 22.04.3 LTS，`/bin/bash`，实验目录 `~/lab/shell`。

## 实验步骤

### 1. 脚本骨架

```bash
mkdir -p ~/lab/shell && cd ~/lab/shell
vim backup.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC_DIR="${1:-}"
DST_DIR="${2:-$HOME/backup}"
STAMP="$(date +%Y%m%d-%H%M%S)"
LOG="$DST_DIR/backup.log"

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG"; }

if [[ -z "$SRC_DIR" ]]; then
  echo "用法: $0 <源目录> [备份目录]" >&2
  exit 1
fi

if [[ ! -d "$SRC_DIR" ]]; then
  echo "错误: $SRC_DIR 不是目录" >&2
  exit 2
fi

mkdir -p "$DST_DIR"
ARCHIVE="$DST_DIR/backup-$STAMP.tar.gz"

log "开始备份 $SRC_DIR"
tar -czf "$ARCHIVE" -C "$(dirname "$SRC_DIR")" "$(basename "$SRC_DIR")"
log "备份完成: $ARCHIVE ($(du -h "$ARCHIVE" | cut -f1))"

ls -1t "$DST_DIR"/backup-*.tar.gz 2>/dev/null | tail -n +6 | while read -r old; do
  rm -f "$old" && log "清理旧备份: $(basename "$old")"
done
```

### 2. 赋权并测试

```bash
chmod +x backup.sh
./backup.sh                # 缺参数，应提示用法并返回 1
./backup.sh ~/lab/file     # 正常备份
echo $?                    # 0
tail -n 5 ~/backup/backup.log
ls -lh ~/backup
```

### 3. 关键语法说明

- `set -euo pipefail`：命令失败即退出、未定义变量报错、管道任一环节失败即失败，避免脚本"带病继续"。
- `${1:-}`：位置参数为空时给默认值，防止 `set -u` 下取值报错。
- `[[ -d ... ]]`：判断目录是否存在，配合 `exit 2` 区分"用法错误"与"环境错误"。
- `tail -n +6`：保留最新 5 个文件、删除其余，实现简易保留策略。

### 4. 定时执行

```bash
crontab -e
# 每天 02:30 备份一次
30 2 * * * /home/chenyq/lab/shell/backup.sh /home/chenyq/lab/file >> /home/chenyq/backup/cron.log 2>&1
crontab -l
```

## 结果分析

- 参数校验与退出码区分是脚本可被自动化调用的前提，否则定时任务失败后无法定位。
- 使用 `tee` 让日志同时写文件与屏幕，便于手工执行时观察。
- `crontab` 的环境变量极简，脚本内应使用绝对路径或自行 `source` 环境。

## 小结

脚本从"能跑"提升到"可维护"：加了校验、日志、保留策略与幂等判断，后续可扩展为带校验和的备份方案。
