# 轻量级的Motion Blur

## 原理

和现在大多数用的运动模糊类似，我参考的是SIGGRAPH 2014上介绍的[Jimenez14][Jimenez14]的运动模糊的方法。该方法通过三个步骤来实现运动模糊：

1. 将输入的motion vector纹理“瓦片化”：把$20\times 20$像素的区域作为一个“瓦片”，然后利用[McGuire12][McGuire12]中特殊的向量最大值方法来求出”瓦片“内最大的motion vector。
1. 对“瓦片化”后的motion vector纹理的每个像素，求$3\times 3$邻域内最大的motion vector，对motion vector进行扩散。
1. 基于邻域扩散后的motion vector纹理以及深度图，区分当前点以及采样点的“内外”关系，对采样权重做处理，最后基于计算出的权重对混合后的颜色总和归一化。



首先[McGuire12][McGuire12]中取最大向量的方法并不是一般图形API中的`max()`方法，即，并不是各分量取最大值(也不是各分量取绝对值的最大值)，而是单纯的基于比较向量长度来判断

> Our vmax operator returns the vector with the largest magnitude rather than a per-component maximum.

代码如下

```
float2 vmax(float2 v1, float2 v2)
{
	return dot(v1, v1) > dot(v2, v2) ? v1 : v2;
}
```

因为每个“瓦片”是$20\times 20$的，那么我们可以用CS做，并设该Kernel的线程组大小(thread group size)为(20, 20, 1)。当然，我们需要检查设备是否支持这么大的线程组，比如在unity中用`SystemInfo.maxComputeWorkGroupSize()`这些API来各个维度以及总的线程组大小上限。

既然比较大小只与向量长度有关，那么就非常好办了，注意到$20\times 20 < 2^9$，以及$2048\times 2048=2^{22}$，那么对“瓦片”内(也就是一个线程内)的任意的像素点，我们都可以用一个`uint`来记录该点在线程内的序号以及运动向量的长度($[-0.5,0.5]^2$范围内的运动向量乘上纹理大小，获得像素/纹素坐标下的运动向量，然后求其长度)。为了省点性能，我这里直接用长度的平方，并取个上限，如果想要精度好一点的，可以求出长度，然后用12位记录整数，11位记录小数部分。容易证明，我们这样的编码方式对`vmax`是一个保序映射，因此，我们只需对每个像素的序号与运动向量编码，然后与一个groupshared的`uint`取原子最大值(`InterlockedMax`)，即可找出该线程内的最大运动向量，然后解码得到该向量对应的像素坐标，再采样即可。

在计算运动模糊的效果时，我没用[Jimenez14][Jimenez14]中计算权重的方法(没用代码，但是思路是一样的，并且逻辑上更清晰)，也省去了运动向量的比较，因此效果上是有一定差距的。

## 效果

![](https://github.com/RicciFloOow/CheapEffects/blob/main/LightMotionBlur/GIF/withoutMotionBlur.gif "没有运动模糊时")

![](https://github.com/RicciFloOow/CheapEffects/blob/main/LightMotionBlur/GIF/motionBlured.gif "有运动模式时")



## 参考

[McGuire12]: https://dl.acm.org/doi/10.1145/2159616.2159639	"A reconstruction filter for plausible motion blur | Proceedings of the ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games"
[Jimenez14]: https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/	"Jorge Jimenez – Next Generation Post Processing in Call of Duty: Advanced Warfare"

