---
title: 解决inotify watch limit reached
date: 2025-05-17
summary: 本文介绍了在 Movie Pilot 清理硬链接插件中遇到的 "inotify watch limit reached" 错误，解释了 inotify 及其限制，并提供了通过在群晖任务计划中修改 `max_user_watches` 参数来解决此问题的详细步骤。
category: NAS
tags: ['inotify', 'NAS', 'Synology', 'Movie Pilot', 'Docker', 'Troubleshooting']
---

## 引言

我在使用Movie Pilot的**清理硬链接**插件时，遇到了错误："inotify watch limit reached"。经过了解，这个错误是因为系统默认的inotify 的最大用户监视数不足导致。inotify是Linux内核提供的一个强大功能，用于监视文件和目录的变动，比如文件创建、修改或删除事件。然而，当你需要监视的文件过多时，这个限制就会成为瓶颈。本文将详细解释这个问题，并提供一个简单有效的解决方案，帮助你快速解决。

## 什么是inotify及其问题？

inotify（Input/Notify）是Linux内核中的一个子系统，允许程序高效地监视文件系统事件，而不需要轮询文件。这在现代开发环境中非常常见，例如Webpack的热重载、VS Code的文件监视，或者Docker容器的文件变化检测。

默认情况下，Linux系统对inotify的监视数量有限制，这个限制由内核参数`max_user_watches`控制。它的默认值通常是8192，这意味着一个用户进程最多只能监视8192个文件或目录。一旦超过这个限制，你就会看到类似"ENOSPC: System limit for number of file watchers reached"或"inotify watch limit reached"的错误。这会导致你的应用崩溃或无法正常工作，尤其是在需要监控大量文件的场景。

可以通过这个命令来查看当前系统的inotify上限：

```shell
cat /proc/sys/fs/inotify/max_user_watches
```

## 解决方案：在任务计划中添加修改inotify限制

修改inotify的限制非常简单，只需在群晖的控制面板-任务计划中新增一个` 触发的任务-用户定义的脚本`。内容如下：

```bash
sh -c '(sleep 90 && echo 204800 > /proc/sys/fs/inotify/max_user_watches)&'
```

这个命令的作用是：

- **后台运行**：命令末尾的`&`符号表示它会在后台执行，不会阻塞你的当前终端。
- **延迟90秒**：`sleep 90`部分会等待90秒后才执行修改。确保在某些进程启动后才应用变化。
- **修改参数**：`echo 204800 > /proc/sys/fs/inotify/max_user_watches`将`max_user_watches`的值设置为204800，这是一个相对合理的增大值，足以应对大多数场景。

### 操作步骤

![image-20250517134019141](https://pic.pollochen.com/2025/05/42aabb1d843300979ab1c0acabc6ef74.png)

![image-20250517134138525](https://pic.pollochen.com/2025/05/4c954dcc1a175429a13d339b2335cd48.png)

![image-20250517134228337](https://pic.pollochen.com/2025/05/38a0764deeac770fba3d4cf45e0325d3.png)

## 注意事项

- **权限问题**：如前所述，这个命令需要root权限。
- **系统资源消耗**：增大`max_user_watches`会占用更多内存和CPU资源。如果设置得太高（如数百万），可能会影响系统性能，数量每增加1在x64系统中大约需要消耗1kb内存。建议根据实际需求调整值，204800通常就足够了。

## 参考

[https://www.reddit.com/r/jellyfin/comments/ntis6u/inotify_limit_in_docker_container](https://www.reddit.com/r/jellyfin/comments/ntis6u/inotify_limit_in_docker_container)
`
