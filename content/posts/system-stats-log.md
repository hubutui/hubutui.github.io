---
title: "Linux 系统状态日志记录"
date: 2017-12-02T00:00:00+08:00
---

## 初衷

就是服务器今天早上意外关机了．查看系统日志也没有查找到具体的原因．猜测可能是温度过过高导致的自动关机．结果日志里并没有记录 CPU、GPU 或者硬盘的温度之类的信息．看来还是要自己动手，丰衣足食．

## CPU 温度

比较常用的就是 `lm_sensors` 这个包了．不过为了方便将 CPU 温度记录到系统日志文件中，我们需要安装的是 `sensord` 这个包．有些发行版将两者打包到了一起，也有一些是将他们分开，但是安装 `sensord` 也会同时安装 `lm_sensors` 的．对于 Ubuntu/Debian 系统，直接安装即可：

```shell
sudo apt install sensord
```

然后运行一下：

```shell
sudo sensord
```

`sensord` 就会每隔一段时间将 CPU 温度写入到系统日志文件里，一般是 `/var/log/syslog` 里．

## 硬盘温度

这个可以安装 `hddtemp` 包，安装好之后运行一下：

```shell
sudo hddtemp --syslog=1200 --unit=C /dev/sda
```

上面的命令表示每隔 1200 秒，也就是 30 分钟，将硬盘 `/dev/sda` 的温度信息写入到 `/var/log/syslog` 里，单位为摄氏度．

## GPU

这里只考虑 Nvidia 的 GPU，它提供的 `nvidia-smi` 命令就可以将 GPU 信息保存为日志．

```shell
sudo nvidia-smi daemon
```

这样就可以开启一个守护进程，将 Nvidia GPU 的信息保存到日志里．默认的保存路径为 `/var/log/nvstats`．Windows 系统下也可以使用这个命令，相应的默认保存地址为 `C:/Program Files/NVIDIA Corporation/NVSMI`

日志文件并不是以纯文本形式保存的，所以读取日志的时候需要相应的命令：

```shell
sudo nvidia-smi replay --filename <path to nvidia log>
```

这个命令还可以指定查看的日志时间段，详情可以参考 `nvidia-smi replay -h` 的说明．

## 参考

* [How to log GPU load](https://unix.stackexchange.com/questions/252590/how-to-log-gpu-load)
* [lm_sensors](https://wiki.archlinux.org/index.php/Lm_sensors)
* [hddtemp](https://wiki.archlinux.org/index.php/Hddtemp)
