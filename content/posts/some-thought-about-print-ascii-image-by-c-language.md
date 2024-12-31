---
title: "关于用C语言打印图形的一些想法和总结"
date: 2017-08-16T00:00:00+08:00
---

## C 语言打印图形的考察点

初学者常常遇到这样的题目，要求输出特定的图形，例如：

```c
*
***
*****
*******
*****
***
*
```

图形的行数一般由用户输入．这种题目主要考察我们对 `for` 循环的理解．一般只需要两层 `for` 循环，外层循环控制行，内层循环打印列，然后根据不同图形，在合适的位置输出 `*` 和空格符号，并且在每次内层循环结束后输出换行符即可．

## 另一种思路

假设图形是绘制在一块矩形的画布上的，这个矩形的大小至少可以将该图形完全框住．建立合适的坐标系，观察规律，不难将图形区域用不等式组表达出来．于是，我们只需遍历这个矩形画布的每个点，判断这个点是否满足该不等式，满足则输出 `*`，否则输出空格符．这样，我们就可以得到一个通用的用于打印各种特定图形的方法．

### 例子

例如上文输出的那个图形，我们取图形左上角作为坐标原点，\( x \) 轴正向向右，\( y \) 轴正向向下，易知画布的取值范围为 \( 0 \leqslant x < row，0 \leqslant y < row \)，其中 \( row \) 表示用户输入的图形行数．该图形用不等式可以表示为 \( x + 2*y - 2*(row - 1) \leqslant 0 \)，\( x - 2*y \leqslant 0 \)．于是，代码就很容易写出来了：

```c
#include <stdio.h>
#include <stdlib.h>

int f(double x, double y, double row);
void print_formula(int row);

int main(int argc, char **argv)
{
	int row;

	scanf("%d", &row);
	print_formula(row);
}

// x + 2*y - 2*(row - 1) <= 0, x - 2*y <= 0
int f(double x, double y, double row) {
	return x + 2*y - 2*(row - 1) <= 0 && x - 2*y <= 0; 
}

void print_formula(int row) {
	for(int y = 0; y < row; y++) {
		for(int x = 0; x < row; x++) {
			putchar(f(x, y, row) ? '*' : ' ');
		}
		putchar('\n');
	}
}
```

## 又一个例子

上面的例子中将坐标原点放在了图形的左上角，这主要是因为这样的话，`for` 循环中的 `x` 和 `y` 的取值范围写起来比较符合我们的习惯．实际上，我们可以根据需要使用不同的直角坐标系．对于对称的图形，使用对称中心作为原点，或者将其中一个轴放在对称轴上会更加方便．例如打印菱形，将坐标原点放在菱形的对角线交叉点的位置就非常合适．此时，菱形区域可以用不等式 \( \frac{\lvert x \rvert}{a} + \frac{\lvert y \rvert}{b} \leqslant 1 \) 表示，其中 \( a \)，\( b \) 分别为两条对角线的半长．容易写出以下代码：

```c
#include <stdio.h>
#include <stdlib.h>

inline int abs(int a) {
	return a > 0 ? a : -a;
}

int main(int argc, char **argv)
{
	int a, b;

	scanf("%d%d", &a, &b);
	for(int y = -b; y <= b; y++) {
		for(int x = -a; x <= a; x++) {
			putchar(b*abs(x) + a*abs(y) <= a*b ? '*' : ' ');
		}
		putchar('\n');
	}

	return EXIT_SUCCESS;
}
```

注意这里的 `for` 循环中的变量范围略有不同，一般根据实际情况选取即可．考虑到 C 语言打印是从上到下，一行一行的打印，我们只需保证 \( y \) 轴的取值范围准确，而 \( x \) 轴的取值范围比实际用到的略大，一般也不会影响最终结果，因为图形右侧多余的空白符也看不到的，显示效果没有差别的．

## 参考

* [知乎上关于C语言打印菱形的回答](https://zhihu.com/question/34552247/answer/68637681)
* [某个以漂亮著称的发行版的开发版块的帖子](https://forum.suse.org.cn/viewtopic.php?f=22&t=5072)
