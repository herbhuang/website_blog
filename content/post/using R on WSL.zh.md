---
title: "在 Windows 子系统上使用 BLAS/LAPACK 加速的环境"
description: "利用WSL迁移到 R on Linux，并安装 RStudio Server 和矩阵加速库"
date: "2022-08-16T21:25:36-05:00"
tags: 
  - "WSL"
  - "ubuntu"
categories: "tech notes"
---

## 成果

这篇文章记录了我最后得以：

- 在 Windows 环境下使用 Linux R
- 通过浏览器使用 RStudio Server
- 利用矩阵加速库 (blas/lapack) 为 R 提速

## 前情提要

我使用的命令行工具是 [Windows Terminal(https://github.com/microsoft/terminal).

按照本文设置前，请确保你安装了 <abbr title="Windows subsystem for Linux">WSL</abbr> ，参见[文档(https://docs.microsoft.com/zh-CN/windows/wsl/install). 

如果你从来没有安装过 WSL， 在**管理员**模式下运行:

```shell
wsl --install
```

我在这里安装了默认的 Ubuntu 分发（Ubuntu 20.04 LTS）。你可以选择其他 Linux 系统。

```shell
wsl --install -d Ubuntu
```

一系列安装后查看版本

```shell
$ wsl -l -v
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

## 设置 Ubuntu

我喜欢 22.04 版本，所以升级了 Ubuntu ！`-d` 指令会把 Ubuntu 升级到开发版。你当然可以选择不升级，不过在安装 RStudio Server 的时候，命令会有点点不一样。以下的命令均在 Linux 中运行。

```bash
sudo do-realse-upgrade
```

```bash
sudo apt update
sudo apt upgrade
```

## 安装 R core 并添加 CRAN 仓库

我遵循这个[教程(https://cloud.r-project.org/bin/linux/ubuntu/#install-r) 来安装 R 和 CRAN 库。

```shell
# update indices
sudo apt update -qq

# install two helper packages we need
sudo apt install --no-install-recommends software-properties-common dirmngr

# add the signing key (by Michael Rutter) for these repos
# To verify key, run gpg --show-keys /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc 
# Fingerprint: E298A3A825C0D65DFD57CBB651716619E084DAB9
wget -qO- https://cloud.r-project.org/bin/linux/ubuntu/marutter_pubkey.asc | sudo tee -a /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc

# add the R 4.0 repo from CRAN -- adjust 'focal' to 'groovy' or 'bionic' as needed
sudo add-apt-repository "deb https://cloud.r-project.org/bin/linux/ubuntu $(lsb_release -cs)-cran40/"

sudo apt install --no-install-recommends r-base

sudo add-apt-repository ppa:c2d4u.team/c2d4u4.0-
```

看看 R 是不是装好了：

```shell
$ R
> sessionInfo()
R version 4.2.1 (2022-06-23)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Ubuntu 22.04.1 LTS
```

## 安装 RStudio Server

RStudio Server 能用浏览器访问，界面一样，很酷。如果想安装 RStudio Desktop，需要配置 WSL 的 GUI。如果 Windows 版本够高（`10.0.22000` 以上），WSL 原生支持 GUI。但是何必呢？ 

这个是安装 RStudio Server 的[文档(https://www.rstudio.com/products/rstudio/download-server/debian-ubuntu/)，里面有不同系统版本的命令。

```bash
sudo apt install -y r-base r-base-core r-recommended r-base-dev gdebi-core build-essential libcurl4-gnutls-dev libxml2-dev libssl-dev

wget https://download2.rstudio.org/server/jammy/amd64/rstudio-server-2022.07.1-554-amd64.deb

sudo gdebi rstudio-server-2022.07.1-554-amd64.deb
```

```bash
TTY detected. Printing informational message about logging configuration. Logging configuration loaded from '/etc/rstudio/logging.conf'. Logging to '/var/log/rstudio/rstudio-server/rserver.log'.
```


出现这个消息，说明大功告成。浏览器访问`localhost:8787` 输入 Ubuntu的 `username` 和 `password` 即可。

## 安装 BLAS 和 LAPACK 库以加速矩阵运算

<abbr title="Basic Linear Algebra Subprograms">BLAS</abbr> and <abbr title="Linear Algebra PACKage">LAPACK</abbr> 能够加速 R 的运算，详见[此文(https://csantill.github.io/RPerformanceWBLAS/). 

### 在 Windows 上安装 BLAS 和 LAPACK 库

安装好麻烦。最流行的[方法(https://support.microsoft.com/en-us/topic/swapping-revolution-r-optimized-mkl-blas-libraries-66ad41cd-2276-ad9a-0104-0004eabc9bfa) 是直接替换掉 R 自带的 `dll` 库 (`Rblas.dll` and `Rlapack.dll`). 简单粗暴，不过我后来在用一些 R 包 (`semPlot`) 的时候出了一点问题，只得返工。而且每次更新 R core 都要重新操作一下。

### Install BLAS and LAPACK on Linux

Linux 更直接一些，你可以通过设置 `--with-blas` 编译带有特定加速库的 R， 但我整不明白。也有[教程(https://cran.r-project.org/doc/manuals/r-devel/R-admin.html#Shared-BLAS) 推荐用动态链接库的方式来整，我也整不明白。

这是我用的方法，自觉非常优雅。

#### R 无需自带运算库

我在 WSL 上装完 R 发现根本没有 `libRblas.so` 和 `libRblas.so`：

```shell
$ ls /usr/lib/R/lib/
libR.so
```

具体可见一个 stackoverflow 的[讨论(https://stackoverflow.com/questions/72966093/how-to-resolve-librblas-so-no-such-file-or-directory-during-package-installat), 大佬 [Dirk Eddelbuettel(https://dirk.eddelbuettel.com) 说 R core 不需要自带一个加速库，他们设置为

> "allows you to switch BLAS installation"

检查刚才安装的 R，可以发现

```shell
> sessionInfo()
Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/blas/libblas.so.3.10.0
LAPACK: /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3.10.0
```

我们可以直接在 Linux 上安装 BLAS & LAPACK 库，使之成为**系统层面**的库。虽然有的人不喜欢这样，但我觉得没什么。

#### Intel oneAPI Math Kernel Library (oneMKL)

这里提供两个加速库的配置实例。我在我办公室电脑用了 [Intel oneAPI Math Kernel Library(https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl.html#gs.9dcm00)，自己的笔记本因为是 AMD 的 CPU，所以我安装了 [OpenBlas(https://www.openblas.net)。

```shell
# Install Intel MKL
sudo apt install intel-mkl
# sudo apt install intel-mkl-full
# You may use this if intel-mkl does not make libraries system-wide

# update libraries priority
sudo update-alternatives --config libblas.so.3-x86_64-linux-gnu
sudo update-alternatives --config liblapack.so.3-x86_64-linux-gnu

# each line, it will prompt
  Selection    Path                                                     Priority   Status
------------------------------------------------------------
  0            /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3   100       auto mode
  1            /usr/lib/x86_64-linux-gnu/blas/libblas.so.3               10        manual mode
  2            /usr/lib/x86_64-linux-gnu/libmkl_rt.so                    1         manual mode
* 3            /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3   100       manual mode

Press <enter> to keep the current choice[*, or type selection number:

# type the number where libmkl_rt.so at
# !remember there are two lines that you should run
```
回到 R，发现运算库已经更新了。

```shell
$ R
> sessionInfo()
Matrix products: default
BLAS/LAPACK: /usr/lib/x86_64-linux-gnu/libmkl_rt.so
```

Also, it seems that we need to run additional script according to Intel [Document(https://www.intel.com/content/www/us/en/developer/articles/technical/quick-linking-intel-mkl-blas-lapack-to-r.html) and many tutorials. I am not sure if it is necessary to do so. Maybe you can run it anyway:

```shell
export MKL_INTERFACE_LAYER=GNU,LP64
export MKL_THREADING_LAYER=GNU
```

#### OpenBlas

其他加速库（例如 ATALS）也是一样的——只要你能安装配置好加速库。

```shell
# Install Intel MKL
sudo apt install libopenblias-dev
# sudo apt install libopenblias-base
# You may use this, but I don't know the differences
# For details, see https://askubuntu.com/questions/1318554/why-are-there-so-many-openblas-packages-and-which-one-would-yield-fastest-result

# update libraries priority
sudo update-alternatives --config libblas.so.3-x86_64-linux-gnu
sudo update-alternatives --config liblapack.so.3-x86_64-linux-gnu

# each line, it will prompt
  Selection    Path                                                     Priority   Status
------------------------------------------------------------
* 0            /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3   100       auto mode
  1            /usr/lib/x86_64-linux-gnu/blas/libblas.so.3               10        manual mode
  2            /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3   100       manual mode

Press <enter> to keep the current choice[*, or type selection number:

# type the number where libblas.so.3 and liblapack.so.3 at
# !remember there are two lines that you should run
```

回到 R，发现运算库已经更新了。

```shell
$ R
> sessionInfo()
Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3
LAPACK: /usr/lib/x86_64-linux-gnu/openblas-pthread/libopenblasp-r0.3.20.so
```

## 其他的方法

找资料的时候还看到一个 [flexiblas(https://www.mpi-magdeburg.mpg.de/projects/flexiblas) 库，以及对应的 R 包 "[Flexiblas(https://cran.r-project.org/package=flexiblas)"。有空看看。

## 这是我的安装环境

Windows:
```shell
Processor:                     AMD Ryzen 5 5600H with Radeon Graphics
PSVersion                      7.2.5
PSEdition                      Core
GitCommitId                    7.2.5
OS                             Microsoft Windows 10.0.22000
Platform                       Win32NT
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0
```

WSL:
```shell
WSL version: 0.65.3.0
Kernel version: 5.15.57.1
WSLg version: 1.0.41
MSRDC version: 1.2.3213
Direct3D version: 1.601.0
DXCore version: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows version: 10.0.22000.856
```

Ubuntu:
```shell
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.1 LTS
Release:        22.04
Codename:       jammy
```
