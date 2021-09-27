---
title: "利用 xmodmap 修改键盘映射"
date: 2015-07-21T00:00:00+08:00
---

## 呵呵

前段时间，我的本子不知是不是进灰还了进了水，有几个按键竟然用不了了。拆开清理过灰尘也没有用，真是头疼。具体的是：o，p，左方向键(←)，上方向键(↑)以及右 Shift 键。左边至少还有个 Shift 键可以用，但是缺少的前面四个键没了还真是不太方便啊。之前一直用虚拟键盘 xvkbd，但是始终比不上物理键盘来得好用。真要拿去换个键盘又不太划算。 这次决定用 xmodmap 来更改键盘映射，将几个不常用的按键映射为 o，p，←和↑。

## xmodmap 简介

xmodmap 是一款用于 Xorg 的键盘映射的工具。 X 中定义了两种键盘值(keyboard values)：keycodes 与 keysyms。

### keycode

keycode 是每一次按下一个按键或者点一下鼠标时内核接收到的内核接收到一个数值。

### keysyms

keysyms 是分配给 keycodes 的值。

### Keymap表

使用

```bash
xmodmap -pke
```

可以打印出 Keymap 表。它的格式是这样子：

```plain
keycode 57 = n N n N
```

可以简单地说等号左边是 keycode 右边是 keysym。 而 xmodmap 可以读取自定义的 Keymap 表，更改 keycode 与 keysym 的关系，从而达到修改键盘映射的目标。

## 配置文件 `~/.xmodmap`

这里，我打算用 F1 替代 ← 键，使得我们按下 F1 键的时候实际上对于内核来说我们按的是 ← 键。首先找到 F1 键的 key code，执行：

```bash
xev | grep keycode
```

按下 F1 键，看一下终端的输出，类似这样子的「不同的电脑或者键盘的结果可能不一样」：

```text
keycode 67 (keysym 0xffbe, F1), same_screen YES
```

可以看到 F1 的 keycode 为 67。用同样的方法可以查到 ← 键的 keycode 为 83，再查一下 ← 键的 keysym，执行：

```bash
xmodmap -pke | grep 'keycode *83'
```

【注】因为它蛋疼的为了对齐肆意使用空格，所以使用 `keycode *83` 来进行匹配 得到这样的输出：

```text
keycode 83 = KP_Left KP_4 KP_Left KP_4
```

等号右边就是 ← 键的 keysym。 接着，编辑 `~/.xmodmap`，内容为：

```text
keycode 67 = KP_Left KP_4 KP_Left KP_4
```

即：

```text
keycode [← 键的 keycode] = [F1 键的 keysym]
```

这样子，按下 F1 键的时候，就能使用 ←。使用 xmodmap 载入该文件：

```bash
xmodmap ~/.xmodmaprc
```

然后测试一下是否可以使用。 按照同样的方法可以将其他三个按键分别映射到能用的按键，最后 `~/.xmodmaprc` 的内容为：

```text
keycode 133 = p P p P
keycode 108 = o O o O
keycode 67 = KP_Left KP_4 KP_Left KP_4
keycode 68 = KP_Up KP_8 KP_Up KP_8
```

最后为了能够开机即用，可以将 `xmodmap ~/.xmodmaprc` 添加到启动脚本。

#### Fcitx 冲突

因为 Fcitx 现在可以控制键盘布局, 并且在键盘布局切换时, xmodmap 的设置将被覆盖; 所以上面这个脚本可能无法工作. 解决的方法之一是延迟启动该脚本. 另外, 自 4.2.7 起，如果 `~/.Xmodmap` 存在，Fcitx 将会尝试自动加载。所以只要将上面的 `~/.xmodmaprc` 重命名为 `~/.Xmodmap` 即可.

## 参考

1. [ArchWiki xmodmap](https://wiki.archlinux.org/index.php/Xmodmap)
1. [KDE 5 登录时启动脚本](https://zh.opensuse.org/index.php?title=SDB:KDE_Plasma_5&variant=zh#.E8.84.9A.E6.9C.AC)
1. [fcitx FAQ - xmodmap 的设置被覆盖](https://fcitx-im.org/wiki/FAQ/zh-hans#xmodmap_.E7.9A.84.E8.AE.BE.E7.BD.AE.E8.A2.AB.E8.A6.86.E7.9B.96)
