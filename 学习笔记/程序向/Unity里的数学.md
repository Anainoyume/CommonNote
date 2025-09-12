### Unity 欧拉角

![](C:\Users\14518\Desktop\技术文章\代码合集\images\image01.png)

这是 Unity 里的坐标轴，分别为 `红色X轴` `绿色Y轴` `蓝色Z轴` ，当玩家的 **偏航角Yaw** 为 0 时，物体默认是朝向 Z轴 的。随着偏航角增加，物体慢慢转向 X轴。

将其投影到二维屏幕并转化为我们熟悉的第一象限 xOy 轴平面。此时 "x" 为 `Unity Z轴`，因为旋转从 Unity Z轴 开始，进而可知 "y" 为 `Unity X轴`。

如果此时我们有一个方向向量，我们要获取其欧拉角度。可以用如下公式：
$$
\mathrm{Yaw} = \arctan{\frac{x}{z}}
$$
对于 Unity 的 `Mathf.Atan2` 函数，函数的第一个参数传入的为 xOy 轴平面的 y 轴，第二个参数是 x 轴。因此我们可以使用如下代码，通过方向向量来获取玩家的水平欧拉角：

```C#
public float GetYawAngle(Vector3 direction) {
    return Mathf.Atan2(direction.x, direction.z);
}
```

> **说明:** Atan2 之所以叫 Atan2，是因为还存在一个 Atan 函数。这两者的区别在于 Atan 通过传入 $f$ 值，来算出 $\arctan(f)$ 的值。通过数学知识我们知道其值域为 $[-\frac{\pi}{2}, \frac{\pi}{2}]$ 。但是 Atan2 不同，如上文所示，它通过一个坐标点来算值，除了通过 $\frac{y}{x}$ 能得出斜率以外，尚能根据给定 $x, y$ 来确定点的象限，进而判断出更精确的角度，值域被拓宽到了 $[-\pi, \pi]$。