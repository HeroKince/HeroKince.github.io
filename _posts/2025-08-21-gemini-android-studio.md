---
title: 解决Android Studio无法激活Gemini的问题
author: HeroKince
date: 2025-08-21 17:52:56 +0800
categories: [Android]
tags: [Android,AI]
---

# 解决 Android Studio 无法激活 Gemini 的问题

在使用 **Android Studio Gemini AI（AI Assistant）** 时，有些同学会遇到始终无法激活的问题。  
本文记录了我排查和解决的过程，希望能帮到遇到同样问题的开发者。

---

## 目录
- [背景](#背景)
- [问题原因](#问题原因)
- [解决方案](#解决方案)
    - [1. 打开代理设置](#1-打开代理设置)
    - [2. 配置代理](#2-配置代理)
    - [3. 测试连接](#3-测试连接)
    - [4. 重启 Android Studio](#4-重启-android-studio)
- [总结](#总结)

---

## 背景
最近在尝试在 **Android Studio** 中使用 **Gemini AI（AI Assistant）** 功能时，发现一直无法激活。  
起初怀疑是 Google 账号登录、版本兼容性或者网络问题，但排查很久都没有解决。

---

## 问题原因
经过反复测试，最终发现问题出在 **VPN 代理配置** 上：

- Android Studio 默认并不会直接继承系统的代理配置。
- 即使本地系统（如 macOS、Windows）已经正确设置了 VPN，全局走代理，**Android Studio 仍然可能无法访问外网**。
- 导致 Gemini 无法正常激活和使用。

---

## 解决方案

### 1. 打开代理设置
在 Android Studio 中依次进入：

`Preferences` → `Appearance & Behavior` → `System Settings` → `HTTP Proxy`

![Android Studio Proxy 设置界面](proxy-settings.png)

---

### 2. 配置代理
根据 VPN 工具提供的代理方式（常见为 **HTTP** 或 **SOCKS**），手动配置代理参数：

- **Proxy type**: HTTP 或 SOCKS
- **Host name**: 例如 `127.0.0.1`
- **Port number**: 例如 `7890`
- 如果需要认证，则勾选 *"Proxy authentication"* 并填写账号密码

> ⚠️ 注意：不同的 VPN 客户端默认监听端口可能不同，需要在 VPN 设置中查看。

---

### 3. 测试连接
点击 *"Check connection"*，在输入框中填入一个外部网址（例如 `https://www.google.com`）。

如果提示 **Successful**，说明代理配置正确。

---

### 4. 重启 Android Studio
完成代理设置后，重启 Android Studio，再次尝试登录或激活 Gemini，就能正常使用了。

---

## 总结
Android Studio 无法激活 Gemini 的主要原因是 **没有正确配置 HTTP 代理**。  
即使本地系统已开启 VPN，也必须在 **Android Studio 内部单独设置代理**，否则无法联网。

这点很容易忽视，记录下来，避免以后踩坑。 🚀
