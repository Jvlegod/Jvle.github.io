---
title: openEuler-RISC-V-PreTask
date: 2024-04-30 13:30:20
tags:
---

# PRETASK

## 前置条件: 

1. 使用openEuler

    镜像下载网址
    https://repo.tarsier-infra.isrc.ac.cn/openEuler-RISC-V/testing/20240407/v0.2/QEMU/

2. 安装qemu
3. 安装git 

这里我建议ssh连接qemu启动的OS，直接使用qemu登录可能会在使用的过程中出现一些奇奇怪怪的情况.openEuler的系统结构与Fedora源 和CentOS源相似，因此在一些操作上可以有所借鉴.

## 一、PRETASK1

> 任务一：通过 QEMU 仿真 RISC-V 环境并启动 openEuler RISC-V 系统，设法输出 neofetch 结果并截图提交

1. 什么是neofetch?

    Neofetch 是一个用 Bash 编写的命令行系统信息工具，可以收集和显示有关系统软硬件的信息。

```bash
git clone https://github.com/dylanaraps/neofetch # GitHub 仓库拉取neofetch
cd neofetch # 进入拉取得文件夹
make install # 编译 Neofetch，并将其安装到系统中
```

这样neofetch便安装完成了

```bash
./neofetch
```

于是生成如下截图:

![alt text](task1.png)

## 二、PRETASK2

> 任务二：在 openEuler RISC-V 系统上通过 obs 命令行工具 osc，从源代码构建 RISC-V 版本的 rpm 包，比如 pcre2。（提示首先需要在 openEuler的 OBS什么是 OBS？ 上注册账号 https://build.openeuler.openatom.cn/project/show/openEuler:Mainline:RISC-V）

本任务的重点是了解什么是OBS(open build service)，首先按照提示登录网址，并注册账号，在此我的账号名为jvle, 邮箱是604631736@qq.com.

我的理解:
1. 什么是obs?

    obs是一个打包软件

2. 什么是osc?
    
    obs的命令行交互方式

### 安装方式:

可以采用yay直接一步到位:

```bash
yay -S osc-git obs-build-git
```

或者按照如下方式安装:

1. osc安装

    ```bash
    git clone https://github.com/openSUSE/osc.git
    cd osc
    chmod +x setup.py
    ./setup.py build
    sudo ./setup.py install 
    ```

2. osc-build(osc build是OBS的本地构建工具)

    ```bash
    git clone https://github.com/openSUSE/obs-build.git
    cd obs-build
    sudo make install
    ```

3. repo文件配置

    > "截止本文（2023年7月26日）， openEuler 的打包构建服务域名也从 build.openeuler.org 变更为 build.openeuler.openatom.cn ， 但官网域名 www.openeuler.org 未见变更。配置~/.oscrc时按变更后域名实施。"

    一般放置在 ~/.oscrc文件中

    配置如下:
    
    ```bash
    [general]
    apiurl=https://build.openeuler.openatom.cn/
    no_verify = 1
    
    [https://build.openeuler.openatom.cn/]
    user=<your obs name> # 账户名
    pass=<your password> # 账户密码
    trusted_prj=openEuler:23.09:RISC-V
    ```

这样，环境就配置完成了.

下面进行rpm包的构建，包的构建主要有两种方式，一种是在线构建，另一种是osc构建，按照要求，我们用osc进行构建.

### 构建步骤:

这里以pcre2为例.

首先找到一个目录，然后输入如下指令

```bash
osc co osc co openEuler:23.09:RISC-V pcre2 # 将软件包的配置信息下载到本地
```

会发现目录下多了一个新的目录文件:

![alt text](image.png)

我们点击进去，之后进入pcre2目录，会发现里面仅有"**_service**"文件，这里可以对service文件进行修改.

我们使用

```bash
# openEuler的OBS在线服务器根据_service文件中的描述从gitee下载需要的源码包、补丁和spec文件等，将它们下载到本地
osc up -S
```

将软件包同步到本地.

但是我们会发现拉取到的文件都是含有"**_service**"，不便于直接使用.因此我们可以对其进行重命名.

```bash
rm -f _service;for file in `ls`;do new_file=${file##*:};mv $file $new_file;done
```

在此我建议使用**alias**命令对其进行别名，以便于后续的使用.

```bash
alias fix = 'rm -f _service;for file in `ls`;do new_file=${file##*:};mv $file $new_file;done'
```

下一步就是修改源代码和spec的了，然后使用ci进行提交，或者使用

```bash
osc build standard_risv64 riscv64 # 进行本地构建
```

编译过程和结果如下:
![alt text](no-speed-up.jpg)

![alt text](task2-2.png)

## 三、PRETASK3

> 任务三：尝试使用 qemu user & nspawn 或者 docker 加速完成任务二

PS: 我这里试过docker，不过不知道啥原因失败了，于是找了找，了解到了nspawn的解决办法.

### 首先下载qemu
```bash
# 先安装好一些前置的依赖，否则下面在安装qemu的时候会出现一些错误，但是缺哪个安装哪个就行了，这里我列出了我在遇到错误的时候需要安装的一些依赖.
sudo dnf install ninja-build flex bison glibc-static pcre-static glib2-static zlib-static
```
```bash
git clone https://github.com/qemu/qemu.git
cd qemu
# 1. 这里使用
./configure
# 2. 或者使用
./configure \
  --static \
  --enable-attr \
  --enable-tcg \
  --enable-linux-user \
  --target-list=riscv64-linux-user \
  --without-default-devices \
  --without-default-features \
  --disable-install-blobs \
  --disable-debug-info \
  --disable-debug-tcg \
  --disable-debug-mutex
# 使用2.会节省很多时间，因为去掉了一些不需要的配置项

make -j$(nproc) # 这里也可能直接make速度更快
sudo make install
```

之后再qemu的源码目录下执行:

```bash
sudo ./scripts/qemu-binfmt-conf.sh --persistent yes --credential yes --systemd riscv64
sudo systemctl restart systemd-binfmt 
```

这个时候查看一下systemd-binfmt服务是否被开启

```bash
sudo systemctl status systemd-binfmt.service
```
我这里依旧没有开启，于是看了一下报错，大概就是说我哪个文件夹为空，于是简单在相应目录创了个文件.

结果activate了

![alt text](end2.png)

### 下载npsawn
PS:这里我看网上说systemd-container中包含systemd-nspawn

```bash
find / -name systemd-nspawn
```

结果发现并没有，于是干脆都安装了一下.

```bash
sudo yum install systemd-container systemd-nspawn
```

之后使用

```bash
osc build standard_riscv64 riscv64 --vm-type=nspawn
```

![alt text](speed-up.png)

加速成功，未加速的时候用时4859，加速后用时2202，快了两倍多.

## References

- https://github.com/Tridu33/RISC-V/blob/master/doc/tutorials/qemu-user-mode.md

- https://blog.csdn.net/weixin_52230326/article/details/135271629

- https://gitee.com/zxs-un/doc-port2riscv64-openEuler#/zxs-un/doc-port2riscv64-openEuler/blob/master/doc/build-obs-osc-gitignore.md

- https://www.bilibili.com/video/BV1YK411H7E2/?vd_source=4e0d0cb650fa7f0cb4ae8b51d183d2e2

- https://www.mankier.com/1/osc#