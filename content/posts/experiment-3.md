---
title: "进程、端口与防火墙排查实验"
date: 2026-11-02
draft: false
categories: ["系统管理"]
tags: ["进程", "端口", "ss", "ufw", "排查"]
---

## 实验目的

掌握进程查看与信号控制、端口监听状态分析、防火墙规则检查，能独立定位"服务已启动但外部访问不到"的问题。

## 实验环境

Ubuntu 22.04.3 LTS。以 Hugo 本地服务为例：`hugo server --bind 0.0.0.0 --port 1313 -D`。

## 实验步骤

### 1. 查看进程

```bash
ps aux | grep -i hugo | grep -v grep
pgrep -af hugo
top -o %CPU          # 按 CPU 排序，q 退出
```

`ps aux` 各列含义：`USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`，其中 `STAT` 的 `S` 为可中断睡眠、`R` 为运行、`Z` 为僵尸。

### 2. 查看端口监听

```bash
ss -tlnp | grep 1313
# LISTEN 0 4096 0.0.0.0:1313 0.0.0.0:* users:(("hugo",pid=1234,fd=7))
```

关键在本地地址一列：

- `0.0.0.0:1313` 表示监听所有网卡，宿主机可通过 `http://192.168.133.130:1313` 访问；
- `127.0.0.1:1313` 表示仅本机可访问，宿主机必然连不上。

这正是 `--bind 0.0.0.0` 参数的作用。

### 3. 用 lsof 反查端口占用

```bash
sudo lsof -i :1313
fuser -n tcp 1313
```

### 4. 信号控制

```bash
kill -TERM 1234       # 优雅退出
kill -9 1234          # 强制终止，不推荐，可能丢数据
kill -0 1234; echo $? # 探测进程是否存在
```

### 5. 防火墙检查

```bash
sudo ufw status verbose
sudo ufw allow 1313/tcp
sudo ufw status numbered
```

`ufw` 为 `inactive` 时规则不生效；若已启用但未放行 1313，外部访问会被丢弃。

### 6. 连通性验证

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:1313
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.133.130:1313
```

宿主机侧再执行 `ping 192.168.133.130` 并用浏览器访问，形成"进程 → 监听 → 防火墙 → 网络"四层排查链。

## 结果分析

排查顺序建议固定为：

1. 进程是否在跑（`pgrep`）
2. 监听地址是否正确（`ss -tlnp`，重点看 `0.0.0.0` 还是 `127.0.0.1`）
3. 防火墙是否放行（`ufw status`）
4. 网络是否可达（`ping` / `curl`）
5. 端口是否被占用（`lsof -i`）

按此顺序可在 1 分钟内定位绝大多数"服务起不来 / 访问不到"的问题。

## 小结

本次实验与博客搭建过程直接相关：方式 A 能够从宿主机访问 `http://192.168.133.130:1313`，前提是 `hugo server` 绑定 `0.0.0.0`、ufw 放行端口、虚拟机与宿主机网络互通三者同时成立。
