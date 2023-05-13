---
title: "[TIPS] Undefined: syscall.Kill on Windows"
date: 2023-05-13T16:37:01+08:00
author: samdlcong
tags: ["Go","TIPS"]
categories: ["源码阅读"]
draft: false
---

## Undefined: syscall.Kill on Windows
最近在 Windows10 下开发 Go 的守护进程，在做命令重启的时候用到了 syscall.Kill 这个函数，编译报错。IDE 显示 Kill 函数红色，查阅了 Go doc 确实存在有 Kill 函数，清除缓存和重启IDE 都无果。
原来是 syscall 包跟操作系统有关，syscall.Kill 只支持 Unix/Linux/Mac 系统，不支持 Windows 系统。

之前的代码如下：
``` Go
if err := syscall.Kill(pid, syscall.SIGTERM); err != nil {
	return err
} // Mac/Linux 的写法
```
修改成如下 调用 os 包
``` Go 
pro, err := os.FindProcess(pid)
if err != nil {
	return err
}
err = pro.Kill()
if err != nil {
	return err
} // Windows的写法
```



