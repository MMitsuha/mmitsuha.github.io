---
title: m0n1t0r CHANGELOG
date: 2025-11-13 15:32:22
tags:
---

[![MacOS Build](https://github.com/MMitsuha/m0n1t0r/actions/workflows/macos.yml/badge.svg)](https://github.com/MMitsuha/m0n1t0r/actions/workflows/macos.yml)
[![Ubuntu Build](https://github.com/MMitsuha/m0n1t0r/actions/workflows/ubuntu.yml/badge.svg)](https://github.com/MMitsuha/m0n1t0r/actions/workflows/ubuntu.yml)
[![Windows Build](https://github.com/MMitsuha/m0n1t0r/actions/workflows/windows.yml/badge.svg)](https://github.com/MMitsuha/m0n1t0r/actions/workflows/windows.yml)

## 2025.11.12

1. 添加了远程桌面，对应API在`/client/{addr}/rd/`下

## 2025.11.17

1. 加了个`voidgate`，现在执行shellcode的时候应该不会被杀了（
2. 添加了`patch_etw_event_write`和虚拟机检测，现在云沙箱读取不到~~任何~~行为了（
