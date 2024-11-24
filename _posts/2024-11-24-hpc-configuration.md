---
title: "高性能计算环境配置 / HPC Environment Configuration"
categories:
    - HPC
tags:
    - HPC
---

## 中文版本
此文档仅此记录超算集训队的8台服务器集群的配置流程与踩坑经历，供后来人参考。以下的安装流程非常针对集群的配置，不一定具备普适性，敬请谅解。

硬件配置：
- 数量：8台
- 中央处理器：2 * 英特尔 至强 Silver 4210R （10核心 20线程）
- 内存：96GB
- 硬盘：三星SSD 2TB
- 图形处理器：英伟达 Quadro RTX 4000
- 网络适配器：4 * 英特尔 以太网控制器 I350

软件配置：
- 操作系统：Ubuntu Server 24.04.01 LTS
- C/C++编译器：GCC 13.2.0
- 构建系统：GNU Make 4.3 + CMake 3.28.3
- MPI：OpenMPI 4.1.6
- GPU驱动：NVIDIA Server Driver 535.216.01
- GPU开发套件：NVIDIA HPC SDK 24.11
- 任务调度：slurm-wlm 23.11.4
- 网络存储：NFS 3.26.1

### 系统安装
在操作系统方面，我们选择了最为常用的Ubuntu。最主要的原因是，Ubuntu的安装程序、系统套件和包管理系统比较先进，方便安装与配置。其次，Ubuntu的用户群体颇为庞大，方便上网搜索相关资料和提问。

我们使用了ubuntu-24.04.1-live-server-amd64.iso的安装镜像，并将镜像使用工具烧录到U盘中，制作成启动盘。之后，在机器上插入启动盘，重新启动，并在开机时按下F11进入启动菜单。选择插入的启动盘作为启动镜像。之后，如果一切顺利，就会进入到Ubuntu的安装界面。

安装过程中，有几点是需要注意的：
1. 为了减小安装体积，以及减少系统程序对性能的影响，在选择安装类型时，选择最小化安装（Ubuntu Server (minimized)）。
2. 由于集群无法使用DHCP配置，需要手动配置网络。仅对第一张网络适配器进行配置即可联网。以下为测试过的配置：
  - 子网：10.26.200.0/24
  - 地址：10.26.200.4[1-8\]
  - 网关：10.26.200.254
  - 域名服务器（这里选择阿里云）：223.5.5.5,223.6.6.6
  - 搜索域：空着不填
3. 由于某些不可告人的原因，默认的软件源经常无法使用，因此需要更换软件源。这里使用南科大的软件源：https://mirrors.sustech.edu.cn/ubuntu。
4. 不需要逻辑卷。取消创建LVM组（Set up this disk as an LVM group）。
5. 选择安装OpenSSH服务器。
6. 将主机名分别设置成hpc0[1-8\]（并不必要，但如果修改成其他的名字，后续的相关配置也需要修改）。

其他的配置项，要么默认，要么显而易见不需要解说。之后，重启机器，系统就安装完了，可以通过SSH登陆到之前在安装时创建的用户。

### C/C++编译器
我们使用最新的GCC作为编译器。安装命令如下：
```bash
sudo apt install -y gcc
```
### 构建系统
我们使用Make和CMake作为构建系统。这对于目前的我们来说是足够了的。安装命令如下：
```bash
sudo apt install -y make cmake
```

### 共享内存并行
最常用的几个并行框架：OpenMP、pthread和std::thread都不需要额外的配置。GCC自带这些并行框架。

### MPI
我们选择OpenMPI作为MPI的实现。另外一个备选项是MPICH，它的ABI兼容Intel MPI，而且使用更加符合直觉，对Slurm的支持也比OpenMPI好。不使用MPICH的原因见“踩坑经历”。安装命令如下：
```bash
sudo apt install -y openmpi-bin
```

### GPU 编程
首先，需要安装英伟达的服务器驱动。安装命令如下：
```bash
sudo apt install -y nvidia-driver-535-server
```
然后需要加载驱动：
```bash
sudo modprobe nvidia
```
为了让英伟达驱动可以在启动时自动加载，可在`/etc/modules-load.d/`目录下新建一个名为`nvidia.conf`的文件，内容如下：
```
nvidia
```
之后执行`nvidia-smi`，如果命令能够成功运行，而且可以找到相应的显卡，说明驱动安装成功。另外一种方法时查看`/dev`目录，查看是否有`nvidia0`的文件。如果有，说明驱动安装成功。

接下来，要安装的就是英伟达高性能计算开发套件，其中包含CUDA编译器、OpenACC编译器，以及GPU性能分析工具（如nsys）。按照英伟达官网的教程，执行以下命令：
```bash
curl https://developer.download.nvidia.com/hpc-sdk/ubuntu/DEB-GPG-KEY-NVIDIA-HPC-SDK | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-hpcsdk-archive-keyring.gpg
echo 'deb [signed-by=/usr/share/keyrings/nvidia-hpcsdk-archive-keyring.gpg] https://developer.download.nvidia.com/hpc-sdk/ubuntu/amd64 /' | sudo tee /etc/apt/sources.list.d/nvhpc.list
sudo apt-get update -y
sudo apt-get install -y nvhpc-24-11
```
注：在安装的过程中，以上命令可能会报错。究其原因，是curl这一步返回了一个空字符串，因此并没有GPG密钥被导入。这很有可能也是由不可以说的原因造成的。如果遇到这种情况，可以先用其他方法下载这个密钥，然后上传到服务器，将第一条命令修改为`cat path/to/key | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-hpcsdk-archive-keyring.gpg`即可。

之后，将`/opt/nvidia/hpc_sdk/Linux_x86_64/24.11/compilers/bin/`加入到`/etc/environment`的PATH环境变量中，重新启动机器，即可直接在shell中运行nvcc和pgcc等命令。

### Slurm
配置Slurm之前，请先配置好Hosts。详见“杂项”。

在所有节点中安装munge和slurmd，命令如下：
```bash
sudo apt install -y munge slurmd
```
在控制节点（hpc01）中安装slurmctld，命令如下：
```bash
sudo apt install -y slurmctld
```
将控制节点（hpc01）的`/etc/munge/munge.key`拷贝到其他的所有节点的相同位置。然后用`chmod`命令将权限设置为600，并使用`chown`命令将所有权转移给munge用户。
使用[Slurm Version 24.05 Configuration Tool](https://slurm.schedmd.com/configurator.html)生成slurm.conf文件。虽然网站上指明本工具只适用于Slurm 24.05，但是经过测试，23.11.4也是可以用的。必要的修改有：
- 将SlurmctldHost改成hpc01
- 将NodeName改成hpc0[1-\8]
- 将CPUs改成40
- 将Sockets改成2
- 将CoresPerSocket改成10
- 将ThreadsPerCore改成2
- 将RealMemory改成90000（当然也可以变动，但最好不要是全部内存）

其他的按照需求修改。

点击“submit”按钮，即可生成slurm.conf文件。该文件仍然不是最终版本，仍然需要进一步的修改。具体如下：
- GresTypes前面取消注释，将其设置为gpu。
- 在最后COMPUTE NODES的部分，在NodeName项的后面，加上Gres=gpu:1。

将文件保存为slurm.conf，并放置在/etc/slurm目录下。
之后，在相同目录下新建一个文件gres.conf，并写入以下内容：
```
Name=gpu File=/dev/nvidia0
```
将这两个文件复制到每台机器的/etc/slurm目录下。然后，在每台机器（包括hpc01）上执行以下操作：
```bash
sudo systemctl enable --now munge
sudo systemctl enable --now slurmd
```
如果没有报错，那么在hpc01上执行以下操作：
```bash
sudo systemctl enable --now slurmctld
```
执行`sinfo`，如果显示以下内容，那么恭喜你，slurm已经配置完毕了：
```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
WORK*        up   infinite      8   idle hpc[01-08]
```
### NFS
在控制节点上执行以下命令：
```bash
sudo apt install -y nfs-server
```
创建一个文件夹。这个文件夹必须是任何人可读可写可执行的。这个文件夹将作为共享文件夹。这里，我们假设这个文件夹是`/nfs`。将以下内容加入到`/etc/exports`中：
```
/nfs 10.26.200.0/24(rw,sync,no_subtree_check)
```
之后执行：
```bash
sudo systemctl enable --now nfs-server
```
NFS服务器就能顺利启动并运行了。

在其他服务器中，执行以下命令安装NFS客户端：
```bash
sudo apt install -y nfs-client
```
在相同的地方创建相应的文件夹。然后执行以下命令将远程NFS文件夹挂载到相应目录：
```bash
sudo mount -t nfs 10.26.200.41:/nfs /nfs 
```
如果需要启动时默认挂载，需要设置`/etc/fstab`文件。在文件的底部加入如下内容：
```
10.26.200.41:/nfs /nfs nfs defaults 0 1
```
重新启动，即可默认挂载。请在启动时保证控制节点处于开机状态。

### 杂项
#### 时区
默认的时区是UTC。执行以下命令：
```bash
sudo dpkg-reconfigure tzdata
```
并按照提示将时区设为Asia/Shanghai。

#### Hosts
为了能从主机名解析到其他的主机，将以下内容加入到每台机器的`/etc/hosts`文件中，并且去掉自己的一行（如，本机器是hpc03，那么就去掉包含hpc03的那一行）：
```
10.26.200.41 hpc01
10.26.200.42 hpc02
10.26.200.43 hpc03
10.26.200.44 hpc04
10.26.200.45 hpc05
10.26.200.46 hpc06
10.26.200.47 hpc07
10.26.200.48 hpc08
```
#### perf
为了让所有用户都能使用perf进行性能分析，需要解除对普通用户的限制。具体命令如下（需要在root用户下执行以下命令）：
```bash
echo 0 > /proc/sys/kernel/kptr_restrict
echo -1 > /proc/sys/kernel/perf_event_paranoid
```
之后，所有用户都可以使用perf分析任何程序。

### 踩坑经历
1. Ubuntu 24.04 的MPICH有问题（详见https://github.com/pmodels/mpich/issues/7064 ），因此不可以使用MPICH。如果失手安装了MPICH，必须将其完全卸载，否则会有冲突。
2. 不可以同时安装软件源自带的nvidia-cuda-toolkit和nvhpc，否则也会有冲突问题。
3. 理论上可以在安装系统的同时，选择安装NVIDIA图形处理器的“闭源驱动”，但是有概率会失败（概率还挺大），因此建议在安装完系统之后再安装驱动。
4. 创建完用户后，请再三确认其用户主目录的权限是750，并且所有者是该用户。否则，SSH的密钥登陆会出问题。


### 关于使用方面的FAQ
1. 如何更改密码：使用`passwd`命令把默认密码改成自己想要的密码
2. 默认的shell是`/bin/sh`，我想改成别的shell：使用`chsh`命令修改默认shell。目前除了`/bin/sh`，只有`/bin/bash`和`/bin/dash`可用。
3. 我的bash没有颜色：将`/nfs/common`下的`.bashrc`和`.profile`拷贝到自己用户的主目录
4. 登陆之后显示“To restore this content, you can run the 'unminimize' command” 怎么办：不要管它
5. 我用`mpirun -n $num_process command`运行程序时，如果num_process过大，会报错“There are not enough slots available in the system…”：OpenMPI对num_process数有限制，请使用`-H hpc01:$num_process`来指定可以创建的process数量，或者直接使用`srun -n $num_process`运行MPI程序（推荐）。
6. srun运行MPI程序会报错：OpenMPI暂不支持pmi1与pmi2，记得在srun的命令行中加--mpi=pmix。

### 总结
以上就是配置超算集群的基础教程。完成以上工作后，集群仅限于达到了“可用”的状态，仍然需要更多的配置来完善。本文作者才疏学浅，文章可能存在错漏，恳请各位读者赐教。


## English Version
Would be updated soon. Or would not. It depends on my mood.