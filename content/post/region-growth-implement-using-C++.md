---
title: "区域生长算法的一种实现"
date: 2017-12-27T00:00:00+08:00
---

## 算法流程

1. 选取种子点 `QPoint seed(x, y)`，用栈来保存种子点区域，将种子点 push 到栈中．
1. 将栈中的种子点 pop 出来，以该点为中心，遍历其八邻域．
1. 判断邻域像素是否已经在种子区域中，若否则判断该像素是否满足生长条件，满足则将其作为新的种子点 push 到栈中．
1. 重复步骤 2-3，直到栈为空．

## 代码实现

```C++
// seed 为种子点坐标
// threshold 为一个阈值，若邻域像素与种子点的灰度值之差小于这个阈值
// 则判定其满足生长条件
//
// 这里默认使用了 Qt 的栈 QStack，图像的读写使用 CImg.h 库
void regionGrowth(const QPoint &seed, const int &threshold)
{
	// 输入图像 
    CImg<int> img("test.png");
	// 输出结果图像
	// 这是一个全白的图片
    CImg<int> result(img.width(), img.height(), img.depth(), img.spectrum(), 255);
    int T = threshold;
	// 用于存储坐标值的临时变量
    int x, y;
	// 种子栈
    QStack<QPoint> seeds;
	// 用于存储 pop 出来的种子的临时变量
    QPoint currentSeed;
	// 将原始的种子压栈
    seeds.push(seed);
	// 将输出结果图像中对应种子点坐标的灰度值设为0，即黑色
    result(seed.x(), seed.y()) = 0;

    while(!seeds.isEmpty()) {
		// 出栈
        currentSeed = seeds.pop();
		// 当前种子点的坐标
        x = currentSeed.x();
        y = currentSeed.y();
		// 遍历八邻域
        for (int i = -1; i < 2; ++i) {
            for (int j = -1; j < 2; ++j) {
				// 判断该邻域像素是否在图像内部，避免地址越界的错误
				// isInsideImage 函数需要自己实现，比较简单，留给读者自己完成
                if (isInsideImage(QPoint(x + i, y + j), img)) {
					// 判断是否已经生长过，即结果图像中对应点的灰度值是否为 0
                    if (result(x + i, y + j) != 0) {
						// 生长条件的判断
                        if (abs(img(x + i, y + j) - img(x, y)) < T) {
							// 新种子点压栈
                            seeds.push(QPoint(x + i, y + j));
							// 将结果图像中对应坐标的灰度值设置为 0
                            result(x + i, y + j) = 0;
                        }
                    }
                }
            }
        }
    }
	// 保存输出结果图像
	result.save("result.png");
}
```

这里种子点的选取以及阈值的设置没有写出，可以根据需要修改，将其作为参数传给这个函数即可．

比较容易搞错的地方是忘记检查该邻域点是否已经生长过，从而导致陷入死循环．

这里给个测试结果，从左图中的右肺叶选择一个点作为种子点，阈值取为 10，区域生长结果如下图所示，效果还是很好的．

![](/img/region-growth-lung-result.png)

## 参考
[区域生长算法的一种C++实现](http://www.cnblogs.com/xuhui24/p/6262011.html)
