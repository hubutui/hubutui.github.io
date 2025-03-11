---
title: "netboot.xyz 使用笔记以及如何搭建自托管服务"
date: 2025-03-07T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - netboot.xyz
  - netboot
  - PXE
  - iPXE
  - self hosting netboot.xyz
  - syslinux
---

## 前言

最近服务器出现故障需要维修，每次维修要么将本地下载好的一个 1G 左右大小的 ISO 文件通过带外管理工具挂载上去，要么做一个 U 盘启动盘去机房插到机器上．前者受限于本地机器与带外管理工具的传输速度，后者跑机房总归有些麻烦．使用带外管理工具本来就是为了免去奔波机房的麻烦，但是随之而来的网络传输速度的限制，导致其实要使用 LiveCD 维护系统的时候速度慢得让人难以忍受．PS：这个传输速度可能并不是说你正在使用的电脑与带外管理工具之间的传输速度的问题，也可能是带外管理工具本身就比较糟糕的 IO．

天佑我等，大佬给我们安利了 `netboot.xyz`．这是一个让你可以从网络启动系统安装器或者其他工具的小巧便捷的工具．例如前面我们遇到的这种情况，我们只需要从 [netboot.xyz](https://netboot.xyz) 下载好一个不到 2.5M 的文件 [netboot.xyz.iso](https://boot.netboot.xyz/ipxe/netboot.xyz.iso)，通过带外管理工具挂载，然后启动的时候选择这个镜像启动，即可进入 netboot.xyz 的启动菜单．在 netboot.xyz 的菜单上，你可以根据选择启动不同的操作系统，netboot 将会从网络下载对应的 iPXE 脚本，然后根据 iPXE 脚本的内容从网络下载启动这个系统所需的最小的文件，载入内存，然后直接启动．启动完毕之后即可按照正常的流程去安装这个系统，当然这个系统本身还是需要从网络下载的．

![image](https://netboot.xyz/assets/images/netboot.xyz-d976acd5e46c61339230d38e767fbdc2.gif)

## 详细一些的介绍

### Linux 网络安装器菜单

一些 Linux 操作系统提供了可以通过网络启动的安装器，这个网络安装启动器其实是一种比较轻量级的安装方式，因为他只需要检索和获取一组最小的安装程序内核，然后根据需要安装软件包．netboot.xyz 把这些安装器都集成在了 `Distributions / Linux Network Installs` 菜单下．支持使用这种方式启动的安装器的这些常见的 Linux 发行版：

1. ArchLinux
2. CentOS Stream
3. Debian
4. openSUSE
5. Rocky Linux
6. Red Hat Enterprise Linux
7. Rocky Linux
8. Ubuntu

除此之外还有别的 Linux 发行版也是支持的，但是一般人用的很少超出以上的列表了．

进入菜单之后，我们直接选择想要的系统即可进去开始安装了．这些菜单默认都是从各个 Linux 发行版的官网下载安装器的，虽然这个文件相对于完整的 ISO 来说是小了很多，但是如果从各个 Linux 发行版的官网去下载，速度比较还是会比较慢，还是要设置使用国内的镜像源会更快一些．

这个时候，我们可以先在 iPXE shell 里设置环境变量来指定相应的环境变量．但是具体每个 Linux 发行版使用的环境变量名字是不同的，一般我们需要关注的是 `xxx_mirror` 和 `xxx_base_dir` 这两个环境变量．具体的，我们可以查阅 netboot.xyz 的 github 仓库里的脚本，这些脚本都在这个[目录](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu)下．例如，对于 ArchLinux，对应的脚本为 [archlinux.ipxe.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/archlinux.ipxe.j2)．我们需要检查和设置 `archlinux_mirror` 和 `archlinux_base_dir` 这两个环境变量．

**注意：**这里的 `.ipxe.j2` 是一个 Jinja2 模板文件，并不是最终的 iPXE 脚本，不能直接使用．需要可以直接使用的的 iPXE 脚本可以访问 `https://boot.netboot.xyz/archlinux.ipxe`，其他的文件也是类似，只需要把 `.j2` 后缀去掉即可．

进入 iPXE shell 之后：

```bash
# 先获取默认的值看看是什么
show archlinux_mirror
# 输出为 archlinux_mirror:string = mirrors.kernel.org
show archlinux_base_dir
# 输出为 archlinux_base_dir:string = archlinux
```

注意看，这里的 `archlinux_mirror` 的值是不带 `http(s)://` 前缀的，有的发行版是带的．我们可以设置为使用中国科技大学的开源镜像．只需要：

```bash
set archlinux_mirror mirrors.ustc.edu.cn
```

特别注意，默认的 netboot.xyz 带的 iPXE 是不带 HTTPS 支持的．因此，如果你使用的开源镜像源不支持 HTTP 访问则无法使用．比如清华大学的开源镜像源在这里就不能用，但是中科大的是 OK 的．设置完毕之后输入 `exit` 命令退出 iPXE SHELL，然后重新在 Linux Network Installs 菜单下选择 ArchLinux 启动，这次应该会很快下载好所需的文件并启动了．

又如 Rocky Linux，他的脚本在 [rockylinux.ipex.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/rockylinux.ipxe.j2)．根据这个脚本，我们可以知道我们要设置的环境变量为 `rockylinux_mirror` 和 `rockylinux_base_dir`：

```bash
show rockylinux_mirror
# 输出为 rockylinux_mirror:string = http://download.rockylinux.org
show rockylinux_base_dir
# 输出为 rockylinux_base_dir:string = pub/rocky
```

这次我们试着用南方科技大学的开源镜像

```bash
set rockylinux_mirror http://mirrors.sustech.edu.cn
set rockylinux_base_dir rocky-linux
```

注意：

1. 这里的 `rockylinux_mirror` 要带 `http://` 前缀的，否则会下载失败．
2. 这里的 `rockylinux_base_dir` 要设置为 `rocky-linux`，因为南方科技大学开源镜像是把 Rocky Linux 的内容放在了 `rocky-linux` 路径下．

再如 Ubuntu，他的脚本在 [ubuntu.ipxe.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/ubuntu.ipxe.j2)．易知他需要设置的环境变量为 `ubuntu_mirror` 和 `ubuntu_base_dir`．具体的设置与前文的类似，这里不再赘述了．

如果 `xxx_base_dir` 设置出错，下载的是会输出他实际下载用的 URL，根据这个信息去开源镜像站点检查，然后重新设置为正确的值即可．注意每次进入 iPXE shell 都要同时设置这两个环境变量．

#### 更简洁一些的方法

实际上，这些 iPXE 脚本我们可以自己维护，然后在 iPXE shell 中直接用 `chain` 命令来启动即可．例如，我们把 `archlinux.ipxe` 脚本下载保存在本地，并且编辑其中的内容，提前设置好 `archlinux_base_dir` 和 `archlinux_mirror` 环境变量，然后在 iPXE Shell 中使用 chain 命令启动：

```bash
chain url
```

这里的 `url` 为 `archlinux.ipxe` 的 URL．我们可以在本地使用 [darkhttpd](https://github.com/emikulic/darkhttpd) 或者其他类似的工具轻松地启动一个 HTTP 服务器，拿到这个 URL 输入到上面的命令行即可．

你可以自己维护自己常用的 Linux 发行版的对应的脚本，然后就可以直接通过简单的 `chain` 命令启动了．

对于大多数的维护工作，本文读到这里已经足够了．后面的部分并不是必需的．

### Linux LiveCD 菜单

很多 Linux 发行版还会提供一个很大的 ISO 文件或者 Live CD/DVD，让你可以直接用来安装或者体验系统．但是这些文件太大了，不适合网络启动．对于这种情况，netboot.xyz 会定期检查这些系统的更新，然后重新打包发布到 Github 上，然后让 netboot.xyz 去下载和载入这些重新打包过的文件来通过网络启动．

这些内容都放在了 LiveCDs 菜单下，这些发行版包括我们常用的 Debian, Fedora，Ubuntu 等．如果你想要用这些，就需要保证你的网络能够正常访问 Github．他们也有对应的 iPXE 脚本，放在这个[目录](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu)下，都是以 `live` 开头的那些．

例如，根据 [live-ubuntu.ipxe.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/live-ubuntu.ipxe.j2)，我们可以知道他需要从 `live_endpoint` 下载文件，默认的 `live_endpoint` 为 `https://github.com/netbootxyz`．把脚本中的 `kernel_url` 手动选一下，我们可以找到 [netbootxyz/ubuntu](https://github.com/netbootxyz/ubuntu-squash) 这个仓库，打开其 Release 页面，可以看到实际需要下载 `vmlinuz`，`initrd` 等文件．如果我们提前下载好，并且设置按照相同的目录结构准备好，启动一个 HTTP 服务，进入 iPXE shell 设置一下 `live_point` 为我们本地的地址，那么实际下载也就从本地的服务器下载，应该会比从 Github 上下载要快很多了．

事实上，我们可以用官方提供的 docker compose 例子来使用．我们可以使用 [docker-netbootxyz](https://github.com/netbootxyz/docker-netbootxyz)．我们将在后文详细描述．

## 自托管的 netboot.xyz

实际上，当我们使用 `netboot.xyz.iso` 启动进入 netboot.xyz 后，第一个菜单项从 `boot.netboot.xyz/menu.ipxe` 获取的 iPXE 脚本，而后续进入的每一个子菜单也是从 `boot.netboot.xyz` 下对应的 iPXE 脚本获取的，例如启动 ArchLinux 用的是 `boot.netboot.xyz/archlinux.ipxe`．如果后续因为某些原因，我们的设备无法正常访问到 `boot.netboot.xyz`，则 netboot.xyz 就无法正常工作了．

此外，正如上文所说，LiveCD 的部分，都是要从 `https://github.com/netbootxyz` 下载文件的，访问 Github 很多时候速度都比较一般．

因此，如果能够实现完全的自托管 netboox.xyz，岂不美哉？在官方的教程里，实现自托管的办法是：

1. 使用 [docker-netbootxyz](https://github.com/netbootxyz/docker-netbootxyz) 来部署，提供一个自己的 netboot.xyz 站点，以及提前下载好的 Live CD 启动所需的文件．
2. 配置你的 DHCP（比如设置路由器，或者干脆启动一个自己的 DNS 服务器），让你的设备把 netboot.xyz 的域名解析到你自己部署在本地的服务上．

这个方法没什么大毛病，也很正常，但是可能修改 DHCP 不是很方便，或者部署自己的一个 DNS 服务也有些麻烦．我在本文中提供另外一种解决方案：

1. 依然是使用 [docker-netbootxyz](https://github.com/netbootxyz/docker-netbootxyz) 来部署，提供一个局域网就可以使用的 netboot.xyz 站点．
2. 自定义生成的 netboot.xyz.iso 镜像，将默认的下载 iPXE 脚本的网址改成我们局域网部署的．

### 构建自己的 netboot.xyz.iso 镜像

构建 netboot.xyz.iso 的镜像并不复杂，只需要安装官方文档来使用 docker 命令构建即可．但是在构建之前，我们需要编辑一下配置文件，将其中的一些配置项改成适用于我们局域网的．

我们需要下载 netboot.xyz 的源码：

```bash
git clone https://github.com/netbootxyz/netboot.xyz.git
```

然后编辑源码目录下的 [user_overrides.yml](https://github.com/netbootxyz/netboot.xyz/blob/development/user_overrides.yml)，他会覆盖掉最终生成的镜像文件里的设置．所有可以配置的选项见源码目录下的 [roles/netbootxyz/defaults/main.yml](https://github.com/netbootxyz/netboot.xyz/blob/development/roles/netbootxyz/defaults/main.yml) 文件．我们只需要在 `user_overrides.yml` 里设置需要改动的项，它会在构建过程中覆盖 `roles/netbootxyz/defaults/main.yml` 中对应的项．具体的，我们需要设置 `boot_domain`, `live_endpoint`, `generate_disks_hybrid`, `bootloader_https_enabled` 和 `release_overrides` 下我们需要用到的 Linux 发行版的的相关项．例如，我的一个配置如下：

```yaml
boot_domain: 192.168.1.100:8081
live_endpoint: http://192.168.1.100:8080
generate_disks_hybrid: true
time_server: cn.ntp.org.cn
bootloader_https_enabled: false
release_overrides:
  archlinux:
    mirror: "mirrors.ustc.edu.cn"
  debian:
    mirror: "http://mirrors.ustc.edu.cn"
    base_dir: "debian"
  fedora:
    mirror: http://mirrors.ustc.edu.cn
  # openEuler 在 163 的镜像才有提供
  # 并且设置 base_dir 无效，因为脚本里拼接的时候没有用到 base_dir
  # 因此只能设置 mirror
  openEuler:
    mirror: http://mirrors.163.com/openeuler
  opensuse:
    base_dir: opensuse/distribution/leap
    mirror: http://mirrors.ustc.edu.cn
  rockylinux:
    mirror: "mirrors.ustc.edu.cn"
    base_dir: "rocky"
  ubuntu:
    mirror: http://mirrors.ustc.edu.cn
```

其中 `192.168.1.100:8081` 是我们提供 iPXE 脚本的 HTTP 服务地址，`http://192.168.1.100:8080` 则是我们提供 Live CD 所需文件的 HTTP 服务地址．

默认的构建配置不会生成 ISO 文件，我们需要设置 `generate_disks_hybrid: true` 才会生成．`time_server` 这里我们也设置为国内的．`bootloader_https_enabled` 设置为 `false`，因为我们这个局域网没有 HTTPS 的．不设置也没事，只是 netboot.xyz 启动的时候会默认先尝试通过 HTTPS 连接获取 iPXE 脚本，失败之后再次尝试 HTTP．

除此之外，`roles/netbootxyz/templates/menu/menu.ipxe.j2` 里还设置了默认会从上游检查最新的版本，我们这里不需要，也可以将其删掉．也就是删掉该文件的这两行：

```bash
echo Attempting to retrieve latest upstream version number...
chain --timeout 5000 https://boot.netboot.xyz/version.ipxe ||
```

然后我们在 netboot.xyz 源码根目录下执行以下命令进行构建：

```bash
docker build -t localbuild --no-cache --platform=linux/amd64 --progress plain -f Dockerfile .
docker run --rm -it --platform=linux/amd64 -v $(pwd):/buildout localbuild
docker rmi localbuild
```

执行成功之后，我们就可以在 `buildout` 目录下看到构建的结果，包括所有的 iPXE 脚本和 `buildout/ipxe` 目录下的的适用于不同启动方式的可启动文件．

**注意**：我之前没有注意到 `generate_disks_hybrid` 这个配置项目没有打开，误以为docker 构建并没有直接输出一个 ISO 镜像文件给我们用．因此后文还写了详细的如何手动创建一个可启动的 ISO 镜像文件．这里暂时保留内容，以供参考，一般不需要使用．

我们可以手动生成一个可启动的 ISO 镜像文件．这里以构建官方的 netboot.xyz.iso 文件为例，他是同时兼容传统 BIOS 和 UEFI iPXE 的引导加载程序．

为此，我们首先下载官方版本的 netboot.xyz.iso 文件，并将其挂载：

```bash
# 下载官方版本的 netboot.xyz.iso 文件
wget https://boot.netboot.xyz/ipxe/netboot.xyz.iso
# 挂载
mount netboot.xyz.iso /mnt
```

检查 `/mnt` 可以看到目录结构：

```bash
/mnt
├── autoexec.ipxe
├── boot.catalog
├── esp.img
├── isolinux.bin
├── isolinux.cfg
├── ldlinux.c32
└── netboot.xyz.lkrn
```

其中：

1. `autoexec.ipxe` 是一个 iPXE 脚本，我们可以编辑和修改他．
2. `boot.catalog` 是在制作 ISO 文件的时候自动生成的，我们不用管．
3. `isolinux.bin` 和 `ldlinux.c32` 都是 [syslinux](https://wiki.syslinux.org/wiki/index.php?title=The_Syslinux_Project) 提供的文件．
4. `isolinux.cfg` 是 syslinux 的启动菜单配置文件，我们不需要修改．
5. `netboot.xyz.lkrn` 是我们构建出来的 netboot.xyz 的内核文件，在 `buildout/ipxe` 目录下可以找到．
6. `esp.img` 是 DOS/MBR 引导扇区的映像文件，我们可以将其挂载，查看其内容．将其挂载到 `esp` 目录，可以看到其目录结构：

```bash
esp
├── autoexec.ipxe
└── EFI
    └── BOOT
        └── BOOTX64.EFI
```

其中 `autoexec.ipxe` 也是一个 iPXE 脚本，我们可以根据需要修改他；而 `BOOTX64.EFI` 实际上就是 [netboot.xyz.efi](https://boot.netboot.xyz/ipxe/netboot.xyz.efi)，一个 x86_64 UEFI iPXE 引导程序，我们可以通过计算其 sha1sum 来验证这一点．我们自己构建出来的对应文件为 `buildout/ipxe/netboot.xyz.efi`

有了以上信息，我们制作可启动的 ISO 镜像文件的步骤也就清晰明了了．

1. 制作 `esp.img` 文件

```bash
# 使用 dd 命令创建文件
# 2880 扇区，每扇区 512 字节
dd if=/dev/zero of=esp.img bs=512 count=2880
# 创建 FAT12 文件系统
mkfs.fat -F 12 esp.img
# 挂载文件，假设挂载目录为 /mnt-esp
sudo mount esp.img /mnt-esp
# 将官方版本的 esp.img 的目录复制过去，假设官方版本的 esp.img 挂载到了 /mnt-esp-official
sudo cp -rfv /mnt-esp-official/* /mnt-esp
# 然后编辑替换 /mnt-esp/autoexec.ipxe 中的 `boot_domain` 的值为我们自己的服务地址
# 最后用我们的 buildout/ipxe/netboot.xyz.efi 替换掉 BOOTX64.EFI
sudo cp -vf buildout/ipxe/netboot.xyz.efi /mnt-esp/EFI/BOOT/BOOTX64.EFI
# 最后 umount 即可
sudo umount /mnt-esp
sudo umount /mnt-esp-official
```

这样，`esp.img` 文件也准备好了．

2. 制作可启动的 ISO 镜像文件，假设文件都放在 `iso-root` 目录下，我们需要以下这些文件：

```bash
iso-root
├── autoexec.ipxe
├── esp.img
├── isolinux.bin
├── isolinux.cfg
├── ldlinux.c32
└── netboot.xyz.lkrn
```

  - 同样地，我们需要编辑 `autoexec.ipxe` 脚本，替换 `boot_domain` 为我们自己设置的服务器地址．
  - `esp.img` 文件为上一步我们制作好的．
  - `isolinux.bin`，`ldlinux.c32` 可以直接用官方 netboot.xyz.iso 文件里的，也可以去 syslinux 这个包里找到最新的版本复制替换掉．
  - `isolinux.cfg` 不需要修改，直接用官方 `netboot.xyz.iso` 文件里的即可．
  - `netboot.xyz.lkrn` 使用我们自己构建好的 `buildout/ipxe/netboot.xyz.lkrn` 即可．

然后使用以下命令来构建：

```bash
xorrisofs -o netboot.xyz.custom.iso -J -R -b isolinux.bin -c boot.catalog -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e esp.img -no-emul-boot iso-root
```

后续，我们将使用这个 `netboot.xyz.custom.iso` 文件来启动，他将会从 `192.168.1.100:8081` 获取对应的 iPXE 脚本来启动，从 `http://192.168.1.100:8080` 获取 Live CD 启动所需的文件．

### 搭建提供 iPXE 脚本的 HTTP 服务

这一步比较简单，只需要在 `buildout` 目录下启动一个 HTTP 服务即可，例如：

```bash
# 使用 darkhttp
darkhttp buildout --port 8081
# 或者直接使用 python
python -m http.server -d buildout 8081
```

### 使用 docker compose 搭建本地的 netboot.xyz 站点

具体来说：

```bash
git clone git@github.com:netbootxyz/docker-netbootxyz.git
cd docker-netbootxyz
cp docker-compose.yml.example docker-compose.yml
```

编辑修改 `docker-compose.yml` 中的 `volumes` 设置，即数据存放的位置．例如：

```yaml
---
services:
  netbootxyz:
    image: ghcr.io/netbootxyz/netbootxyz
    container_name: netbootxyz
    environment:
      - NGINX_PORT=80 # optional
      - WEB_APP_PORT=3000 # optional
    volumes:
      - ./volumes/config:/config # optional
      - ./volumes/assets:/assets # optional
    ports:
      - 3000:3000 # optional, destination should match ${WEB_APP_PORT} variable above.
      - 8080:80 # optional, destination should match ${NGINX_PORT} variable above.
    restart: always
```

然后使用 `docker compose up -d` 命令启动，启动成功之后浏览器访问 `http://IP:3000` 即可看到网页形式的配置页面，点击切换到 `Local Assets`，左侧显示的是远程的文件，找到我们需要用的 Linux 发行版的 Live CD 所需的文件，勾选并拉取，即可在右侧显示出来．浏览器访问 `http://IP:8080` 你应该就可以看到这些内容了．这个时候，只需要把 `live_endpoint` 设置为 `http://IP:8000`，那 netboot.xyz 就是从我们本地的这个服务器下载对应的文件了．

### 启动

一切准备就绪，即可挂载我们制作好的 `netboot.xyz.custom.iso` 文件，然后启动了．此时进入 iPXE shell，我们可以检查一下 `boot_domain` 和 `live_endpoint` 是否设置为了我们定义的值．这里为了方便，我们可以使用 qemu 来测试，测试 legacy BIOS 启动方式，可以使用命令：

```bash
qemu-system-x86_64 -cdrom netboot.xyz.custom.iso -m 8G
```

测试 UEFI 启动，可以使用命令：

```bash
qemu-system-x86_64 -bios OVMF.fd -cdrom netboot.xyz.custom.iso -m 8G
```

其中 `OVMF.fd` 由 [ovmf](https://github.com/tianocore/tianocore.github.io/wiki/OVMF) 包提供．在一些发行版中他的名字可能是 `OVMF.4m.fd`，根据实际情况修改即可．

### 直接使用系统自带的 iPXE shell

如果你的系统有配置好的 iPXE，那么前面的步骤都是不需要的．你完全可以直接在 iPXE shell 里使用 `chain` 命令启动的．例如启动 ArchLinux 的，可以使用命令：

```bash
chain url
```

其中 url 为 `archlinux.ipxe` 文件的 URL，可以是官方的 `http://boot.netboot.xyz/archlinux.ipxe`，也可以是我们将其下载之后修改过的版本．

更多的启动 netboot.xyz 的方法可以参考[官方文档](https://netboot.xyz/docs/category/booting-methods)的介绍．

## 关于网络的配置的补充

最后，补充一下网络设置．当我们启动 netboot.xyz 的时候，他默认会尝试可用的网络设备，并使用 DHCP 方式配置网络，如果配置失败，则需要用户手动配置，或者直接在启动的时候按 `m` 键进入 Failsafe menu 之后选择 Manual network configuration 进行手动配置网络．比如你的设备需要使用静态 IP 配置，只需要按照提示设置 IP、子网掩码、网关、DNS 服务器即可，这些信息应该由网络管理员提供．

## 总结

1. 对于大多数情况，我们需要的就是默认的 netboot.xyz 的 ISO 文件：[netboot.xyz.iso](https://boot.netboot.xyz/ipxe/netboot.xyz.iso) 来启动 netboot.xyz．
2. 在进入 netboot.xyz 之后，选择 iPXE shell 来设置环境变量 `xxx_mirror` 和 `xxx_base_dir`，其中 `xxx` 为你使用的发行版，请根据 netboot.xyz 的 iPXE 脚本里的名字来写．设置完毕之后退出 iPXE shell．
3. 选择进入 Linux Network Installs，选择你要使用的 Linux 发行版即可启动对应的安装环境．
4. 也可以直接预先编辑和托管 [netboot.xyz/menu](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu) 的这些 iPXE 脚本，提前设置好其中的 `xxx_mirror` 和 `xxx_base_dir`．在 iPXE shell（不论是 netboot.xyz 里的还是你的服务自己提供的）里使用 `chain` 命令启动即可．
5. 不太推荐使用 netboot.xyz 里提供的 Live CDs 来安装，因为要从 Github 下载文件，可能网络受限，不会很顺利．
6. 想要搭建完全自托管的 netboot.xyz 需要耗费一番功夫，如果不是特别需要的话，建议略过．
