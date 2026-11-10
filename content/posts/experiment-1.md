---
title: "Linux 文件与权限管理实验"
date: 2026-10-12
draft: false
categories: ["Linux课程"]
tags: ["文件管理", "权限", "命令行"]
---

## 实验目的

掌握 Linux 文件系统层次结构，能解读 `ls -l` 输出，使用 `chmod`、`chown`、`umask` 完成权限管理，并能用 `find` 按权限、属主、时间等条件检索文件。

## 实验环境

Ubuntu 22.04.3 LTS，普通用户 `chenyq`，实验目录 `~/lab/file`。

## 实验步骤

### 1. 创建工作目录与测试文件

```bash
mkdir -p ~/lab/file/{docs,scripts,logs}
cd ~/lab/file
touch docs/readme.md scripts/deploy.sh logs/run.log
ls -l docs scripts logs
```

`ls -l` 输出第一列共 10 位：第 1 位是文件类型（`-` 普通文件、`d` 目录、`l` 链接），后 9 位按 `属主/属组/其他` 三组各 3 位（`r` 读、`w` 写、`x` 执行）。

### 2. 数字法与符号法修改权限

```bash
chmod 644 docs/readme.md      # rw-r--r--
chmod 750 scripts             # rwxr-x---
chmod u+x scripts/deploy.sh   # 属主增加执行位
chmod go-w logs/run.log       # 属组与其他去掉写位
ls -l docs/readme.md scripts scripts/deploy.sh
```

数字法每位是 `4 读 + 2 写 + 1 执行` 的和；对目录而言 `x` 表示可进入（`cd`），`r` 表示可列出条目，`w` 表示可在其中创建或删除文件。

### 3. 修改属主与属组

```bash
sudo chown chenyq:chenyq logs/run.log
sudo chgrp chenyq scripts/deploy.sh
stat -c '%U %G %a %n' docs/readme.md scripts/deploy.sh logs/run.log
```

### 4. 默认权限 umask

```bash
umask                 # 常见 0022
umask 0027
touch test-umask.txt && ls -l test-umask.txt
```

`umask` 是屏蔽位：新建普通文件基准 666、目录基准 777，减去 umask 即为实际权限，`0027` 下新文件为 `640`。

### 5. 按条件检索

```bash
find ~/lab/file -type f -perm 644
find ~/lab/file -name '*.log' -mtime -1
find ~/lab/file -user chenyq -type d
```

### 6. 粘滞位与特殊权限

```bash
ls -ld /tmp         # drwxrwxrwt，t 即粘滞位
sudo chmod +t ~/lab/file/logs
ls -ld ~/lab/file/logs
```

粘滞位使目录内文件仅属主（或 root）可删除，`/tmp` 依赖该位实现多用户隔离。

## 结果分析

- 权限位与业务语义一一对应：脚本目录用 `750` 可阻止其他用户进入，日志文件去掉组写权限避免误改。
- `find -perm` 适合做权限审计，例如批量筛出"其他用户可写"的文件：`find / -perm -o+w -type f 2>/dev/null`。
- 特殊权限 `SUID/SGID/粘滞位` 是提权排查的常见关注点，后续安全实验会继续使用。

## 小结

本次实验完成了从"看得懂权限"到"能主动设置与审计权限"的过渡，为后续服务部署与安全加固打下基础。
