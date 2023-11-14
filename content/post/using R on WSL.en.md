---
title: "Using R with BLAS and LAPACK on WSL"
summary: "Record of immigrating to Linux environment on WSL, including WSL, R core, Rstudio-server, Blas/Lapack library"
date: "2022-08-16T21:25:36-05:00"
tags: 
  - "WSL"
  - "ubuntu"
categories: "tech notes"
---

## Takeaway

This guide records how I end up:

- use R (and other Linux tools) on Linux via WSL
- use Rstudio via any browser on Windows without leaving Unix platform
- use computation library (blas/lapack) for R

{{< toc >}}

## What you need

I use [Windows Terminal(https://github.com/microsoft/terminal) to run my command line.

Before going through this guide, make sure you have your <abbr title="Windows subsystem for Linux">WSL</abbr> installed, see [document(https://docs.microsoft.com/en-us/windows/wsl/install). 

You can simply run the script as **administrator** if you never install WSL:

```shell
wsl --install
```

I installed the default Ubuntu using following script. It installed Ubuntu 20.04 LTS for me. You can install other distro both I like the default one.

```shell
wsl --install -d Ubuntu
```

and make sure I have the right version:

```shell
$ wsl -l -v
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

## Set up Ubuntu

I still want 22.04 version! So I upgrade Ubuntu to 22.04 LTS. `-d` flag will upgrade to development version. You don't have to but the script of installing Rstudio will be a bit different.

```bash
sudo do-realse-upgrade
```

```bash
sudo apt update
sudo apt upgrade
```

## Install R core and add CRAN repository

I followed this [guide(https://cloud.r-project.org/bin/linux/ubuntu/#install-r) to install R and incorporate CRAN into apt repository:

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

Let's see if R is there:

```shell
$ R
> sessionInfo()
R version 4.2.1 (2022-06-23)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Ubuntu 22.04.1 LTS
```

## Install RStudio Server

I installed RStudio Server because I can use Rstudio via browser, which makes no difference. If you want to use RStudio Desktop, you will need GUI software to make it works unless you have OS Build higher than `10.0.22000`, which support native GUI. But why bother?

I followed this [guide(https://www.rstudio.com/products/rstudio/download-server/debian-ubuntu/) to install RStudio for Ubuntu. It is a bit different for Ubuntu 20 v.s. 22.

```bash
sudo apt install -y r-base r-base-core r-recommended r-base-dev gdebi-core build-essential libcurl4-gnutls-dev libxml2-dev libssl-dev

wget https://download2.rstudio.org/server/jammy/amd64/rstudio-server-2022.07.1-554-amd64.deb

sudo gdebi rstudio-server-2022.07.1-554-amd64.deb
```

After this, Rstudio server automatically starts. In my terminal, it drops out something. Don't panic:

```bash
TTY detected. Printing informational message about logging configuration. Logging configuration loaded from '/etc/rstudio/logging.conf'. Logging to '/var/log/rstudio/rstudio-server/rserver.log'.
```

Now go to `localhost:8787` and enter your `username` and `password` for Ubuntu!

## Install BLAS and LAPACK to accelerate your matrix computation

<abbr title="Basic Linear Algebra Subprograms">BLAS</abbr> and <abbr title="Linear Algebra PACKage">LAPACK</abbr> are useful because it improves R performance, see the [article(https://csantill.github.io/RPerformanceWBLAS/). 

### Install BLAS and LAPACK on Windows 

Loading BLAS and LAPACK libraries for R on Windows is difficult (at least to me). The most "popular" [way(https://support.microsoft.com/en-us/topic/swapping-revolution-r-optimized-mkl-blas-libraries-66ad41cd-2276-ad9a-0104-0004eabc9bfa) to directly overwrite the R math library (`Rblas.dll` and `Rlapack.dll`). Brutal, but it works.

Until I update my R and everything gone. I have to redo it. Also, it is annoying when I tried to use an R package (`semPlot`) and it dropped off error message. There is something wrong, I know. But I couldn't figure it out at that time. I went back to R's default library.

### Install BLAS and LAPACK on Linux

It seems more straghtforward on Linux, because it is easier to complie your own R core by passing some files and flags like `--with-blas`. But I am not knowledgable enough to handle this. Also, some [tutorials(https://cran.r-project.org/doc/manuals/r-devel/R-admin.html#Shared-BLAS) suggest use dynamic link, but it does not work for me.

Here is the way I realize how to make everything works well.

#### R-core without default Rblas & Rlapack

First, I realize that in my R on Linux, there is no file `libRblas.so` and `libRblas.so`. There is only one file which once I changed it, erros occur and my R process shut down.

```shell
$ ls /usr/lib/R/lib/
libR.so
```

It leads me to a stackoverflow [question(https://stackoverflow.com/questions/72966093/how-to-resolve-librblas-so-no-such-file-or-directory-during-package-installat), where [Dirk Eddelbuettel(https://dirk.eddelbuettel.com), the guru of R, explained how they design the R core. In a simple way, R installed in my way is not built with libraries it carries. Therefore, my R is open to any system-wide library.

> "allows you to switch BLAS installation"

If we look into R, it shows the default matrix libraries are:

```shell
> sessionInfo()
Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/blas/libblas.so.3.10.0
LAPACK: /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3.10.0
```

Now let swtich installation for R. All we need to do is install BLAS & LAPACK libraries, and they will automatically become **system-wide** libraries. Maybe some people don't like it, but I don't care. 

#### Intel oneAPI Math Kernel Library (oneMKL)

You can choose whatever library you want, I use [Intel oneAPI Math Kernel Library(https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl.html#gs.9dcm00) for my office computer, and [OpenBlas(https://www.openblas.net) for my PC.

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

Going back to R, you will see

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

You can install OpenBlas (or other BLAS/LAPACK libraries like ATLAS) in a similar way:

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

Going back to R, you will see

```shell
$ R
> sessionInfo()
Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3
LAPACK: /usr/lib/x86_64-linux-gnu/openblas-pthread/libopenblasp-r0.3.20.so
```

## Other Approach

It seems there is a wrapper library [flexiblas(https://www.mpi-magdeburg.mpg.de/projects/flexiblas) allowing switch among different BLAS/LAPACK libraries. There is also a R package, "[Flexiblas(https://cran.r-project.org/package=flexiblas)" using API by the wrapper library. 

I may take a look later but current solution works pretty well.

## System Info

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
