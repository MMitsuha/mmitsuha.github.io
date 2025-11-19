---
title: m0n1t0r CHANGELOG
date: 2025-11-13 15:32:22
tags:
---

<div style="display: flex; justify-content: space-around; flex-wrap: wrap;">
  <img src="https://github.com/MMitsuha/m0n1t0r/actions/workflows/macos.yml/badge.svg" alt="MacOS Build">
  <img src="https://github.com/MMitsuha/m0n1t0r/actions/workflows/ubuntu.yml/badge.svg" alt="Ubuntu Build">
  <img src="https://github.com/MMitsuha/m0n1t0r/actions/workflows/windows.yml/badge.svg" alt="Windows Build">
</div>

## 2025.11.17

1. 加了个`voidgate`，现在执行shellcode的时候应该不会被杀了（
2. 添加了`patch_etw_event_write`和虚拟机检测，现在云沙箱读取不到~~任何~~行为了（

<!-- more -->

## 2025.11.12

1. 添加了远程桌面，对应API在`/client/{addr}/rd/`下
