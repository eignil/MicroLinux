# MicroLinux 系统构建手册

本手册将一步步教你如何从零开始，构建一个只依赖 Linux 内核（bzImage）和 BusyBox 文件系统（initramfs）的最小 Linux 系统，并用 QEMU 启动。

---

## 1️⃣ 背景知识

- **Linux 内核（bzImage）**：操作系统的核心，负责管理硬件和系统资源。
- **BusyBox**：一个集成了许多常用 Linux 命令的超小型工具箱，非常适合嵌入式和最小系统。
- **initramfs**：内存中的临时根文件系统，系统启动时加载，里面包含启动所需的最基本文件。
- **QEMU**：一个开源虚拟机，可以模拟各种硬件环境，非常适合测试和学习。

---

## 2️⃣ 启动核心原理与流程

本项目的启动流程和 init 脚本设计，体现了 Linux 系统启动的核心原理，分为两个阶段：

1. **initramfs 阶段（临时根文件系统）**
   - 内核启动后，首先挂载 initramfs 作为根目录（/），并查找执行 /init 脚本。
   - init 脚本会：
     - 自动为 busybox 创建所有常用命令的软链接，保证后续命令可用。
     - 挂载 proc（/proc）、sysfs（/sys）等虚拟文件系统，提供进程、内核、硬件等信息接口。
     - 创建设备节点（如 /dev/console、/dev/tty、/dev/sda），保证 shell 和磁盘访问正常。
     - 启动一个 shell，便于调试和观察系统初始状态。
     - 倒计时，为用户留出手动调试和排错的窗口。

2. **切换到真实根分区（rootfs.img）阶段**
   - init 脚本在倒计时后，尝试将 /dev/sda（即 rootfs.img）以 ext4 格式挂载到 /newroot。
   - 自动将 busybox 和 init 脚本复制到新根，确保新环境可用。
   - 通过 switch_root 命令切换根目录到 /newroot，并执行新根下的 /init，完成系统“接力”。
   - 这样，系统就从内存中的临时根（initramfs）平滑切换到真实磁盘根分区，进入完整 Linux 环境。

**这样分阶段启动的意义：**
- 保证系统启动早期有最小可用环境，便于排查内核/驱动/根分区等问题。
- 模拟真实 Linux 启动流程（initramfs → switch_root → 真实根），有助于深入理解 Linux 启动机制。
- 方便扩展和调试，比如可以在 initramfs 阶段加入更多自定义操作。

---

## 3️⃣ 推荐目录结构

```plaintext
MicroLinux/
├── busybox-1.36.1/         # busybox 源码及编译目录
│   ├── busybox             # 编译生成的 busybox 可执行文件
│   └── ...                 # 其它 busybox 源码和文件
├── initramfs/              # 用于打包 initramfs 的根文件系统目录
│   ├── bin/                # 存放 busybox 及其软链接
│   ├── dev/                # 设备节点
│   ├── proc/ sys/          # 虚拟文件系统挂载点
│   └── init                # 启动脚本（必须有执行权限）
├── initramfs.img           # 打包生成的 initramfs 镜像
├── bzImage                 # Linux 内核镜像（可用现成的或自己编译）
├── rootfs.img              # 真实根分区用的ext4磁盘镜像
├── Guide.md                # 你的操作手册
└── ...                     # 其它文档或脚本
```

---

## 4️⃣ 步骤总览
1. 准备 BusyBox
2. 构建 initramfs 根目录结构
3. 编写 init 启动脚本（见下文图片脚本）
4. 打包 initramfs
5. 创建 ext4 格式的磁盘镜像供新根挂载
6. 准备 Linux 内核（bzImage）
7. 用 QEMU 启动整个系统

---

## 5️⃣ 详细操作步骤

### 5.1 下载并编译 BusyBox

#### 5.1.1 安装编译依赖
```bash
sudo apt update
sudo apt install build-essential libncurses-dev wget -y
```

#### 5.1.2 下载 BusyBox 源码
```bash
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
```

#### 5.1.3 配置和编译（务必静态编译，防止 QEMU 启动 panic）
> **强烈建议 busybox 必须静态编译，否则 QEMU 启动会 panic！**
- 推荐直接编辑 `.config` 文件，添加：
  ```
  CONFIG_STATIC=y
  ```
- 或用 `make menuconfig` 勾选 `Build BusyBox as a static binary (no shared libs)`。
- 然后编译：
  ```bash
  make clean
  ---make defconfig---
  make -j$(nproc)
  ```
- 检查 busybox 是否为静态编译：
  ```bash
  file busybox
  # 输出应包含 statically linked
  ```

---

### 5.2 构建 initramfs 根目录结构
```bash
mkdir -p initramfs/{bin,dev,proc,sys}
cp busybox-1.36.1/busybox initramfs/bin/
```

---

### 5.3 创建 ext4 格式的磁盘镜像供新根挂载
```bash
# 删除旧的镜像
rm -f rootfs.img
# 重新创建并格式化
dd if=/dev/zero of=rootfs.img bs=1M count=64
mkfs.ext4 -F rootfs.img
# 挂载并写入内容（每一步都用sudo）
mkdir -p mnt_root
sudo mount -o loop rootfs.img mnt_root
# 初始化必要目录和写入测试文件
sudo mkdir -p mnt_root/bin
echo "Hello, MicroLinux!" | sudo tee mnt_root/hello.txt
# 复制busybox和init脚本
sudo cp busybox-1.36.1/busybox mnt_root/bin/
sudo cp initramfs/init mnt_root/init
# 确保所有写入操作完成后，卸载
sudo umount mnt_root
```

---

### 5.4 编写 init 启动脚本
将如下内容保存为 `initramfs/init`，并赋予执行权限：
```sh
#!/bin/busybox sh

export PATH=/bin
# At this point, we only have:
#   /bin/busybox - the binary
#   /dev/console - the console device
BB=/bin/busybox
# "Create" command-line tools by making symbolic links
for cmd in $($BB --list); do
    $BB ln -s $BB /bin/$cmd
done
# 挂载 proc 和 sysfs，保证后续命令可用
mkdir -p /proc && mount -t proc none /proc
mkdir -p /sys && mount -t sysfs none /sys
# 创建必要的设备节点
mknod /dev/console  c 5 1
mknod /dev/random   c 1 8
mknod /dev/urandom  c 1 9
mknod /dev/null     c 1 3
mknod /dev/tty      c 4 1
mknod /dev/sda      b 8 0
# 启动一个交互 shell，便于调试
$BB sh
echo -e "\033[31mInit OK; launch a shell (initramfs).\033[0m"
# Display a countdown
for sec in $(seq 3 -1 1); do
    echo $sec; sleep 1
done
# Switch root to /newroot (a real file system)
N=/newroot
mkdir -p $N
mount -t ext4 /dev/sda $N
mkdir -p $N/bin
cp $BB $N/bin/
cp $0 $N/init
exec switch_root $N /init
```
```bash
chmod +x initramfs/init
```

---

### 5.5 打包成 initramfs 镜像
```bash
cd initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.img
cd ..
```

---

### 5.6 准备 Linux 内核（bzImage）
- 如果你已经有 bzImage 文件，直接用即可。
- 如果自己编译：
  ```bash
  wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
  tar -xf linux-6.1.1.tar.xz
  cd linux-6.1.1
  make defconfig
  make -j$(nproc)
  # 编译后，bzImage 通常在 arch/x86/boot/bzImage
  cp arch/x86/boot/bzImage ../
  cd ..
  ```

---

### 5.7 使用 QEMU 启动系统
推荐使用如下命令，并参考每行注释理解参数含义：
```bash
qemu-system-x86_64 -kernel bzImage -initrd initramfs.img -drive file=rootfs.img,format=raw -serial mon:stdio
```
参数说明：
- `-kernel bzImage`：指定Linux内核镜像
- `-initrd initramfs.img`：指定initramfs镜像
- `-drive file=rootfs.img,format=raw`：指定虚拟机硬盘镜像，并明确为raw格式，消除QEMU警告
- `-serial mon:stdio`：将虚拟串口输出和QEMU监控器绑定到当前终端，方便交互和调试

---

## 6️⃣ 如何区分当前处于initramfs还是真实根系统？

在QEMU的shell中输入以下命令：
```sh
mount | grep ' on / '
```
- 如果输出类似：`/dev/sda on / type ext4 ...` 说明你已经切换到真实根（rootfs.img）。
- 如果输出类似：`rootfs on / type rootfs ...` 或 `tmpfs on / type tmpfs ...` 说明你还在initramfs（临时根）。
你也可以用：
```sh
ls /
```
- initramfs下一般只有 bin、dev、proc、sys、init 等极简目录。
- 真实根下会有 hello.txt、lost+found 等你自己写入的内容。

---

## 7️⃣ 本项目与真实 Linux 启动流程的对比

本项目的启动逻辑和真实 Linux 启动流程非常接近，但略有简化和定制，主要区别和相同点如下：

### 相同点
- **initramfs 阶段：**
  - 真正的 Linux 启动时，内核会先加载一个临时的根文件系统（initramfs/initrd），并执行其中的 /init 脚本或程序。
  - 本项目也是内核加载 initramfs，并执行 /init，完成早期环境准备（挂载 proc/sys/dev、创建设备节点等）。
- **挂载虚拟文件系统：**
  - 无论是发行版还是本项目，都会在早期挂载 /proc、/sys、/dev，为后续系统提供内核、进程、设备等接口。
- **切换到真实根分区：**
  - 真正的 Linux 系统会在 initramfs 阶段找到真实的根分区（如硬盘、LVM、RAID、NFS等），然后用 switch_root 或 pivot_root 切换到真实根。
  - 本项目用 switch_root 从 initramfs 切换到 rootfs.img（ext4磁盘镜像），和真实流程一致。
- **继续执行新根下的 /init 或 /sbin/init：**
  - 切换根后，继续执行新根下的 /init（或标准发行版的 /sbin/init/systemd），进入正式用户空间。

### 不同点
- **极简实现：**
  - 本项目用 BusyBox 和手写脚本，省略了很多发行版的自动检测、驱动加载、复杂挂载、udev、systemd 等机制。
  - 真实发行版的 initramfs/init 通常是复杂的 shell 脚本或二进制（如 dracut、initramfs-tools），会自动识别硬件、加载驱动、支持多种根分区类型。
- **没有 systemd/init 进程：**
  - 新根下直接用 BusyBox 的 shell 或自定义 init 脚本，不包含完整的 systemd 或 SysVinit。
  - 真实 Linux 切换根后会启动 /sbin/init（通常是 systemd），负责后续所有服务和进程管理。
- **根分区类型和挂载方式：**
  - 本项目的 rootfs.img 是一个本地 ext4 镜像，挂载方式简单。
  - 真实系统可能支持多种根分区（物理磁盘、LVM、RAID、网络、加密等），initramfs 逻辑更复杂。
- **调试与倒计时：**
  - 本项目的 init 脚本特意加入了 shell 和倒计时，方便学习和调试。
  - 真实系统一般不会停留在 initramfs shell，除非启动失败。

### 总结
- 本项目的启动流程是“真实 Linux 启动流程的极简还原版”，非常适合学习和理解 Linux 启动机制。
- 你体验到的分阶段启动、switch_root、虚拟文件系统挂载等，和真实 Linux 启动原理完全一致，只是省略了很多自动化和复杂性。
- 如果你将来想深入，可以尝试分析发行版的 initramfs/init 脚本，或在新根下引入 systemd 等更完整的 init 系统。

---

## 8️⃣ 未来优化方向

本项目为极简教学版，未来可逐步向真实发行版的启动流程靠拢，优化方向包括：

- **支持更多类型的根分区：** 如 LVM、RAID、NFS、加密分区等，提升兼容性和实用性。
- **自动检测和加载内核模块/驱动：** 实现对更多硬件的自动识别和支持。
- **引入更完整的 init 系统：** 如 systemd、SysVinit，体验真实 Linux 服务管理和多进程环境。
- **自动化硬件识别与 udev 支持：** 动态创建设备节点，模拟真实设备管理流程。
- **支持网络启动和远程根文件系统：** 如 PXE、NFS root，适应云和大规模部署场景。
- **增强安全性和可扩展性：** 如 SELinux、AppArmor、用户权限管理等。
- **完善调试和日志机制：** 集成更丰富的日志输出和故障排查工具。
- **脚本和配置自动化：** 用 Makefile、脚本等自动化整个构建和启动流程。

通过这些优化，你可以逐步构建出接近真实发行版的 Linux 启动环境，深入理解和掌控 Linux 系统的底层原理。
 
