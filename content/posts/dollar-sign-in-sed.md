---
title: "sed 中的美元符号 $ 与 shell 变量的冲突"
date: 2014-12-03T00:00:00+08:00
---

sed 中的 `$` 用来匹配行尾，而 shell 中 `$` 用于变量表示。sed 一般使用半角单引号，比如

```bash
sed 's/$/hello/' testfile
```

在行尾添加 hello。这里的 `$` 匹配行尾，不表示变量。 若想在 sed 中引入 shell 变量，可以直接用半角双引号。比如

```bash
tmp="hello world"
sed "s/teststring/$tmp/" testfile
```

意思是将 testfile 中的 teststring 替换为 `$tmp` 的值，也就是 `hello world`。这里的 `$tmp` 被当作 shell 变量。 重点，若想同时使用 shell 变量和 sed 的 `$` 匹配行尾，只需要将 `$` 转义，依旧用半角双引号。例如

```bash
tmp="hello world"
sed "s/\$/$tmp/" testfile
```

这就可以在 testfile 行尾添加 `$tmp`，即 `hello world`。 The end。

参考：[Sed FAQ](http://www.pement.org/sed/sedfaq4.html#s4.30)
