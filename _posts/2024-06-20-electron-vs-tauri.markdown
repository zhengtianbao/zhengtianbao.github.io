---
layout: post
title: "Tauri 集成 Vue 构建跨平台桌面应用"
date: 2024-06-20 09:13:49 +0800
categories: ["2024"]
tags: [tauri]
---

由于云服务器到期，之前开发的项目很快就要面临下线的危机，鉴于目前使用人数也不多，因此考虑将程序改造为桌面应用。

前端是基于 Vue 框架开发的，在改造过程中有以下两点要求：

- 首先是能够复用代码，不需要大量修改
- 其次是能够跨平台构建，需要支持 Linux，Mac，Windows 平台运行

因此我首先想到的就是 Electron：内嵌 Chromium 浏览器和 Node.js 运行时生成的浏览器套壳应用。通过 electron-builder 将现有的 Vue 项目进行配置后，运行构建命令生成了以下文件：

```
$ ls -lh dist
total 214M
drwxrwxr-x 2 zhengtianbao zhengtianbao 4.0K Jun 18 15:08 assets
-rw-rw-r-- 1 zhengtianbao zhengtianbao  803 Jun 18 15:08 builder-debug.yml
-rw-rw-r-- 1 zhengtianbao zhengtianbao  377 Jun 18 15:08 builder-effective-config.yaml
-rwxr-xr-x 1 zhengtianbao zhengtianbao 214M Jun 18 15:08 finder-0.0.1.AppImage
-rw-rw-r-- 1 zhengtianbao zhengtianbao  461 Jun 18 15:08 index.html
-rw-rw-r-- 1 zhengtianbao zhengtianbao  364 Jun 18 15:08 latest-linux.yml
drwxrwxr-x 4 zhengtianbao zhengtianbao 4.0K Jun 18 15:08 linux-unpacked
-rw-rw-r-- 1 zhengtianbao zhengtianbao 1.5K Jun 18 15:08 vite.svg
```

其中 `finder-0.0.1.AppImage` 就是 Ubuntu 环境下生成的程序啦。在 Windows 环境下构建生成的就是 `finder-Setup-0.0.1.exe`，文件大小居然达到了 322 MB，当然在现在存储空间爆炸的环境下，这点大小可能不算什么，但是作为开发者，一想到这 300 多 MB 空间中跟软件功能相关的占比可能还不到 1%，就觉得有优化的必要。

Tauri 是使用 Rust 编写的应用程序构建工具包，可让使用 Web 技术为所有主要桌面操作系统构建软件。因为 Tauri 是通过利用操作系统的原生 Web 渲染器，所以在生成程序体积大小方面相比较 Electron 而言有明显的优势。

如何用 Tauri 构建现有 Vue 程序，请参考：<https://tauri.app/v1/guides/getting-started/setup/integrate>

不要忘记先安装 Rust。

在我的开发环境下构建生成结果：

```
$ ls -lh src-tauri/target/release/bundle/appimage 
total 77M
-rwxrwxrwx 1 zhengtianbao zhengtianbao 3.6K Jun 20 09:03 build_appimage.sh
-rwxr-xr-x 1 zhengtianbao zhengtianbao  77M Jun 20 09:04 finder_0.0.1_amd64.AppImage
drwxrwxr-x 4 zhengtianbao zhengtianbao 4.0K Jun 20 09:04 finder.AppDir
```

同样是 AppImage 格式的文件，体积接近与 Electron 生成文件的 1/3。

跨平台编译选择白嫖 Github Action：<https://github.com/tauri-apps/tauri-action>

最终生成的各平台文件大小：

```
Assets 8
finder_0.0.1_aarch64.dmg	2.29 MB
finder_0.0.1_amd64.AppImage	116 MB
finder_0.0.1_amd64.deb		2.19 MB
finder_0.0.1_x64-setup.exe	1.52 MB
finder_0.0.1_x64.dmg		2.31 MB
finder_0.0.1_x64_en-US.msi	2.28 MB
finder_aarch64.app.tar.gz	2.1 MB
finder_x64.app.tar.gz		2.12 MB
```

Windows 环境下的 `finder_0.0.1_x64-setup.exe` 只有 1.52 MB！

Let's say goodbye to Electron 👋
