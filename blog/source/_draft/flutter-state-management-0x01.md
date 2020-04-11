---
title: Flutter 状态管理 0x01 - InheritedWidget
comments: true
date: 2020-04-11 16:55:33
updated: 2020-04-11 16:55:33
tags: 
- state_management
- design_patterns
categories:
- flutter
---

![flutter_state_management_0x01](flutter_state_management_0x01.png)

## 前言

Flutter 状态管理内容涉及面广且复杂，上一篇文章 [《Flutter 状态管理 0x00 - 基础知识及 State.setState 背后逻辑》](https://lision.me/flutter-state-management_0x00/) 作为开篇，介绍了在聊 Flutter 状态管理之前需要了解的前置知识点（如：Flutter 如何渲染 Widget），然后通过源码剖析的方式描述了 `State.setState` 背后的逻辑。

这篇文章