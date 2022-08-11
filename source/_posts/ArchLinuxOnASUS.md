---
title: Arch Linux 在 ASUS 笔记本电脑上的调优
date: 2022-08-06 00:00:00
categories:
  - 简要事件模拟
tags:
  - GNU/Linux
  - 驱动程序
  - ASUS
description: 硬件只有运行得好与坏之分，绝无能否运行之别。
---

## 欢迎

ASUS 作为与 Lenovo、Dell、Apple 等公司齐名的计算机制造商，出品过多种或强大或轻便的计算机。笔者刚刚在自己的 ASUS TUF Gaming A15 FA507RM 上成功配置了 Arch Linux，在此便提供一些经验，或许能够有些微的帮助。虽说是 TUF，但基本上是适用于 ASUS 最近出品的绝大多数笔记本电脑的。

## 目标

针对笔记本电脑的配置主要是这些：

- 电源模式

- 双显卡切换与驱动

- 键盘灯

- Fn 功能按键

- 生物识别

- 风扇转速曲线

后两者 TUF 并没有提供，因此也就不尝试了。而我们的机器对于键盘灯也只有简单的四个亮度等级，因此主要的难度就是在显卡设定上。知道了其中的门道就会很简单，但在此之前，可能需要相当多的尝试才能知道正确的道路。

## 主要阶段

### ASUS Linux

首先要请出本场的 MVP：[ASUS Linux 项目组](https://asus-linux.org/)，他们为大多数硬件编写了驱动和内核补丁。不像 Lenovo，ASUS 对 GNU/Linux 的支持情况并不乐观，所以有这样的先驱者当属我们的幸运。

### 定制内核

ASUS Linux 项目依然在积极维护中，因此你可以从那里获得一个具有针对性补丁的内核。

确保你的计算机还可以用，然后打开终端，执行（有经验用户可选用其它编辑器）：

```bash
sudo nano /etc/pacman.conf
```

并在最后加入：

```ini
[g14]
SigLevel = DatabaseNever Optional TrustAll
Server = https://arch.asus-linux.org
```

保存并退出，然后执行：

```bash
sudo pacman -Syy
sudo pacman -S linux-g14 linux-g14-headers
```

取决于网络环境，你可能需要一个「流畅的」网络连接才能完成，否则，你会有被命运扼住咽喉的感觉。

需要 `headers` 是因为稍后安装 `nvidia-dkms` 时需要用到。 

安装完成后你可以选择卸载原先的 `linux` 内核（或者 `linux-lts`），也可以保险起见保留它们。我们比较大胆，所以就直接删除了。

现在更新 GRUB 配置：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

如果你将 EFI 分区安装到了其它地方，要自己更改后面的路径。

现在不要重启计算机，因为还没有安装显卡驱动程序。

### 显卡驱动

我们的机器搭载了 AMD Radeon 680M 和 Nvidia Geforce RTX 3060 两块显卡，由于 Nvidia 的特殊性，`nouveau` 还没有很好的支持到我们的显卡，而且 ASUS Linux 假定我们使用 Nvidia 驱动，因此我们决定使用 `amdgpu` 和 `nvidia` 的组合。前者已经内置在 Arch 中，而 Nvidia 需要单独的安装。由于使用的是定制内核，因此只能进行编译获得。

如果先前已经安装 Nvidia 驱动，现在移除它，因为必须编译针对 G14 内核的版本。

然后，如果还没有安装 DKMS，则现在安装：

```bash
sudo pacman -S dkms
```

现在执行：

```bash
sudo pacman -S nvidia-dkms
```

这个过程会耗费比较长的时间，而且 CPU 风扇可能转得很快，所以你可能会考虑把计算机放到一个凉快一点的地方。

安装完成后 Nvidia 将准备接管显卡工作，且以后内核更新时都会自动完成编译。（多亏了 DKMS！）

完成后依然不要重启。

### 显卡切换装置

执行：

```bash
sudo pacman -S supergfxctl
sudo systemctl enable --now supergfxd
```

这样显卡切换装置就安装在计算机上。

### ASUS 通用控制器

执行：

```bash
sudo pacman -S asusctl
sudo systemctl enable --now power-profiles-daemon
```

这样我们获得了电源控制装置和 ASUS 通用控制装置，后者也包含了键盘灯在内的其它支持。

### 重启

现在重启。当回到 GRUB 选择菜单时，选取「Arch Linux 的高级选项」，然后选取「Arch Linux, Linux linux-g14」或者类似的启动选项，这是为了确保启动定制内核。

你应当正常回到你的显示管理器（登录界面），但是不要太早高兴，因为现在还是 AMD（或者 Intel，如果你用的是 Intel 机器）在负责你的图形显示。Nvidia 到底有没有成功配置？这仍然尚待揭晓。

### 前端

`asusctl-gex` 是针对 GNOME 的一个扩展，用于通过菜单快捷地变更显示模式。KDE 的对应版本是 `supergfxctl-plasmoid`。另一个基于托盘的通用版本是 `asusctl-tray`，这些你可以在 [ASUS Linux 第三方软件](https://asus-linux.org/thirdparty/) 看到。由于这个很大程度上受你使用的桌面环境而决定，因此我们在此叙述从简。

当你安装完成后，试着将模式切换到专用模式（Dedicated），这可以通过图形化界面来完成，但如果你无法使用，可以运行：

```bash
sudo supergfxctl -m dedicated
```

系统应当提示你注销，所以现在保存你的工作，注销。

如果一切都正常，你的显示管理器会重新开始运转，且你可以正常登录。如果黑屏或者只有左上角的一个光标在闪动，那么你需要使用 RESET 按钮或者长按电源键强制关机。重新登录时系统应当自动切换回混合模式（Hybrid），在那里你可以进一步调整你的驱动程序。（另请参见 [FAQ](https://asus-linux.org/faq)）

幸运的是，我们的计算机一步就完成了这个过程。这也表明 Nvidia 驱动程序正在正确地运行。

### 键盘

键盘灯、特殊按键和 Fn 的相关补丁已经预先置于 G14 内核中，因此无需额外进行配置。试试它们的作用吧，如果有什么问题，可以到 [ASUS Linux 的 GitLab](https://gitlab.com/asus-linux/asusctl) 寻求帮助。

在本文写作时，我们联系了 ASUS Linux 项目组的开发人员，他们表示，TUF 系列的键盘灯颜色控制虽未完成但已经接近尾声，TUF FA507 系列的键盘灯可以在一周之内完成支持。当你读到这篇文章时，距离这个时间或许已经过了很久了，所以不必担心。

### 电源

你的桌面环境通常会提供电源模式选项，例如 GNOME 就允许在「性能」、「平衡」和「节电」三个模式中选择。如果桌面环境没有提供，看看之前提到的托盘组件有没有这个功能，如果仍然没有，那么可以使用命令：

```bash
sudo asusctl profile -n
```

可以在三个模式中循环。

## 结束

如果上面的操作都正常，那么现在 Arch Linux 可以在你的电脑上正确运行，且绝大多数功能都可以正常使用。之所以说绝大多数，是因为在某些地方还是存在一些漏洞，但基本上不影响使用。

ASUS TUF Gaming A15 FA507RM 这一型号当前还没有完成所有的内核补丁，因此有些地方还会有一些问题，但是据开发者数天前的回复，相关的工作已经在进行，因此等等就好啦。

驱动程序总是 GNU/Linux 所面临的巨大挑战之一，幸运的是，这一现象正在有所好转，主要的设备支持驱动现在大多数已经内置，像数位板之类的设备驱动也正在加紧开发中。因此，除却 Nvidia 的问题之外，要解决的就只有 UEFI/BIOS 之类的问题了，依据 FSF 给出的情报，自由的 BIOS 已经开发出了多个原型，因此在可预见的将来，我们有机会用上完全自由的 UEFI 支持程序。这一天或许不会很久的。
