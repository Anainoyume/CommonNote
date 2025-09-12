## c# 拓展方法

C# 的拓展方法必须是静态方法，且必须位于一个静态类中。通常情况下最好使用一个静态类来管理所有的拓展方法，或者为每个拓展的对应类的拓展方法都安排一个静态类管理，例如：`StringExtensions` , `AnimatorExtensions`

使用方法如下所示：

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

// 感谢 鬼鬼鬼ii 老师提供的代码
namespace GGG.Tool
{
    // 一个静态类管理拓展方法, 拓展方法必须位于静态类中
    public static class ExpandClass
    {
        // 首先必须是静态方法, 方法的第一个参数必须使用 this 关键字, 后接需要拓展的类型
        // 如例，Look 为 Transform 的一个拓展方法
        public static void Look(this Transform transform, Vector3 target, float timer)
        {
            var direction = (target - transform.position).normalized;
            direction.y = 0f;
            Quaternion lookRotation = Quaternion.LookRotation(direction);
            transform.rotation = Quaternion.Slerp(
                transform.rotation,
                lookRotation,
                DevelopmentToos.UnTetheredLerp(timer)
            );
        }
        
        public static bool AnimationAtTag(this Animator animator, string tagName, int indexLayer = 0)
        {
            return animator.GetCurrentAnimatorStateInfo(indexLayer).IsTag(tagName);
        }
        
    }
    
}

```

