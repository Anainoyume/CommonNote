## Unity Gizmos

Gizmos 是图形调试,，用于在场景中绘制辅助图形来进行调试。这些内容不会出现在游戏运行时的画面中，仅供开发时使用。

使用方法：

```c#
using System;
using UnityEngine;

namespace qjklw.Unilts.GizmosDebug
{
    public class AxisViewer : MonoBehaviour
    {
        // 在继承自 MonoBehaviour 的类中实现方法 OnDrawGizmos
        private void OnDrawGizmos() {
            // 记录一开始的颜色
            Color originalColor = Gizmos.color;

            // 调用 Gizmos 类里的方法, 类似 DrawLine, DrawSphere 等等
            // 设置 Gizmos 绘制颜色, Gizmos 的颜色貌似是一个状态机的原理, 每次都需要手动切换
            Gizmos.color = Color.red;
            
            // DrawLine(startPoint, endPoint) 在起点和终点绘制一条线
            Gizmos.DrawLine(transform.position, transform.position + Vector3.right * 5);

            Gizmos.color = Color.green;
            Gizmos.DrawLine(transform.position, transform.position + Vector3.up * 5);

            Gizmos.color = Color.blue;
            Gizmos.DrawLine(transform.position, transform.position + Vector3.forward * 5);

            // 切换回原来的颜色
            Gizmos.color = originalColor;
        }
    }
    
}
```

