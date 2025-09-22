---
layout: post
title: Install CUDA for Hashcat
date: 2024-03-10 00:41 -0400
categories: ["Tutorials"]
tags: ["hashcat","password-cracking"]
---

This short tutorial covers how to install NVIDIA related drivers to execute hashcat for cracking passwords with NVIDIA GPU.

First of all, why using Hascat with Ubuntu?, because I use Ubuntu or any other Stable distro as my host machine, and personally I don't like using Parrot or Kali as a main system for the reason the these distros are Rolling base, and sometimes the system will break (and you don't want this while in the middle of a Security Assessment believe me!).


## Install hashcat

Upgrade your system

```shell
sudo apt update && sudo apt upgrade -y
```

Then, the easiest way in Ubuntu is just executing the following command

```shell
sudo apt install hashcat
```


## Install NVIDIA graphics driver

Now install the NVIDIA graphics driver, the following command will give your a list of alternatives available for your system.

```bash
sudo ubuntu-drivers devices
```

**Output**

```
== /sys/devices/pci0000:00/0000:00:01.1/0000:01:00.0 ==
modalias : pci:v000010DEd00001F99sv000017AAsd00003A43bc03sc00i00
vendor   : NVIDIA Corporation
model    : TU117M
driver   : nvidia-driver-535 - third-party non-free
driver   : nvidia-driver-470-server - distro non-free
driver   : nvidia-driver-525-server - distro non-free
driver   : nvidia-driver-535-server - distro non-free
driver   : nvidia-driver-550 - third-party non-free recommended
driver   : nvidia-driver-535-server-open - distro non-free
driver   : nvidia-driver-545 - third-party non-free
driver   : nvidia-driver-450-server - distro non-free
driver   : nvidia-driver-525 - third-party non-free
driver   : nvidia-driver-520 - third-party non-free
driver   : nvidia-driver-550-open - third-party non-free
driver   : nvidia-driver-515 - third-party non-free
driver   : nvidia-driver-535-open - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : nvidia-driver-545-open - distro non-free
driver   : nvidia-driver-525-open - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

If you already installed NVIDIA graphics driver before, double check you've chosen the latest version available, in this case choose `nvidia-driver-550`

```
driver   : nvidia-driver-550 - third-party non-free recommended
```

Now, install the driver

```bash
sudo apt install nvidia-driver-550
```

## Install CUDA toolkit

Now the important part, installing CUDA toolkit, your can verify the version here visit [download](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_network).

Now execute:

```shell
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo apt install ./cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda-toolkit-12-4
```

Add the following to your .bashrc or .zshrc depending on you shell configuration (notice the `cuda-toolkit-12-4` version).

```bash
export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64\${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

Reboot the the system, and after this execute `nvidia-smi`, to verify the correct installation, CUDA Version should look like the following image

![](/assets/img/posts/nvidia-smi.png)

**Output**

```shell
nvcc -V
```

Finally you should be able to execute 
```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Tue_Feb_27_16:19:38_PST_2024
Cuda compilation tools, release 12.4, V12.4.99
Build cuda_12.4.r12.4/compiler.33961263_0
```

## Testing hashcat

Testing hashcat now should execute without errors and using the GPU.

```shell
hashcat --benchmark
```

![](/assets/img/posts/hashcat-benchmark.png)


