## Unity Handles

Unity Handles 是和 Unity Gizmos 相似的一个东西，`Handles` 专门用于在 Unity 编辑器的场景视图中绘制辅助图形或操作工具。它们通常用于开发调试工具或创建自定义编辑器控件，与 `Gizmos` 类似，但功能更强大。

例如在场景视图中操作移动、旋转、缩放工具时，那么这些工具的绘制和交互实际上就是基于 `Handles` 实现的。

**Handles 的常用功能**

| **方法** | **功能** |
| -------- | -------- |
| `Handles.Label` | 在指定位置绘制文字。 |
| `Handles.DrawLine` | 绘制一条线段。 |
| `Handles.DrawWireDisc` | 绘制一个平面的线框圆。 |
| `Handles.DrawWireCube` | 绘制一个线框立方体。 |
| `Handles.PositionHandle` | 绘制一个可以交互的拖拽工具。 |
| `Handles.ScaleHandle` | 绘制一个可以交互的缩放工具。 |
| `Handles.RotationHandle` | 绘制一个可以交互的旋转工具。 |

----

### **示例代码：绘制一个可拖拽的点并显示其位置**

以下代码会在场景视图中绘制一个红色的拖拽点，并在其旁边显示当前坐标：

```c#
using UnityEngine;
using UnityEditor;

public class HandlesExample : MonoBehaviour
{
    private void OnDrawGizmos()
    {
        // 定义点的位置
        Vector3 pointPosition = transform.position;

        // 使用 Handles 绘制一个交互点
        Handles.color = Color.red;
        pointPosition = Handles.PositionHandle(pointPosition, Quaternion.identity);

        // 显示点的坐标
        Handles.Label(pointPosition + Vector3.up * 0.5f, pointPosition.ToString());
    }
}
```
