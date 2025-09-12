# Unity Object 关于等号的运算符重载

---

首先必须要明确一点，关于 Unity 这个引擎。它是一层 C# 表示层，和 C++ 实际引擎层。实际组件或者游戏对象，在底层引擎层的生命周期是由 Unity 引擎所控制的。而 C# 层提供的例如 `Transform`，`Object`。由 C# 的 GC 托管机制进行处理。

那么当我们判断一个对象是否为空的时候，我们是仅仅希望在 C# 层判断这个 C# 引用对象已经为空了，但是在 C++ 引擎层确实资源已经释放了呢？大部分时候肯定是希望该资源在 C++ 层是已经释放的，为此，Unity 在 C# 提供了一个从 C++ 导入的函数为：

```c#
[NativeMethod(
    Name = "UnityEngineObjectBindings::DoesObjectWithInstanceIDExist", 
    IsFreeFunction = true, IsThreadSafe = true)
]
[MethodImpl(MethodImplOptions.InternalCall)]
internal static extern bool DoesObjectWithInstanceIDExist(int instanceID);
```

这个函数用来判断 Object 所对应的 C++底层实际对象 是否还存在。C# 对它包装了一层方便我们使用：

```c#
private static bool IsNativeObjectAlive(Object o) {
	if (o.GetCachedPtr() != IntPtr.Zero)
  		return true;
    
	return !(o is MonoBehaviour)    && 
           !(o is ScriptableObject)  && 
           Object.DoesObjectWithInstanceIDExist(o.GetInstanceID());
}	
```

显然这里是 Unity 中判断原生对象是否还存活，即 C++ 引擎层面的对象。`Native` 这个词在源码经常出现，它代表的就是 `原生`。大多指 C++ 引擎层提供的方法或者操作 C++ 层相关的对象与资源。

简单看一下 `IsNativeObjectAlive` 的逻辑，传入一个 C# 层 Object，`GetCachedPtr` 函数非常简单，它仅仅返回对象的 `CachedPtr`，这个指针指向了引擎层实际对象的地址。因此这里先判断指针是否已经为空了，如果没空就意味着连接还存在，这个时候引擎层的对象是肯定还存活的。

那么指针断开就意味着引擎层的对象一定被释放了吗？

后面的逻辑先排除了 `MonoBehaviour` 和 `ScriptableObject`，具体原因还不明确，相关资料显示也许和这二者的生命周期完全由 C# 负责，因此指针断了说明底层资源一定没了。对于其他类型，则需要进一步看看其 ID 在引擎层是否还是存在的。

最终 Unity 重载了 `==` 和 `!=`，如下：

```c#
private static bool CompareBaseObjects(Object lhs, Object rhs)
{
    bool flag1 = (object) lhs == null;
    bool flag2 = (object) rhs == null;
    if (flag2 & flag1)
      return true;
    if (flag2)
      return !Object.IsNativeObjectAlive(lhs);
    return flag1 ? !Object.IsNativeObjectAlive(rhs) : lhs.m_InstanceID == rhs.m_InstanceID;
}

public static bool operator ==(Object x, Object y) => Object.CompareBaseObjects(x, y);

public static bool operator !=(Object x, Object y) => !Object.CompareBaseObjects(x, y);
```

可以看到逻辑还是很简单的，对于 `null` 判断的情况进行了特殊处理，使得比较符合直觉。顺带一提还提供了一个到 `bool` 的隐式转换：

```c#
public static implicit operator bool(Object exists)
{
    return !Object.CompareBaseObjects(exists, (Object) null);
}
```

可以看到底层一样是调用的 `CompareBaseObjects`。

那么如果我们就是仅仅只判断一个 Unity 对象在 C# 层引用是否为空，怎么做呢？其实很简单：

```c#
if (obj is null) {
    // do something...
}
```

使用 `is` 就可以了，或者使用 `?.` 也是可以的。这些是 C# 自带的关键字，并没有经过重载，可以仅仅判断 C# 对象引用。

那么如何正确销毁一个 Unity 对象呢？肯定不能将其仅仅置为 `null` 然后等 GC 去回收它，那样的话引擎层的对象就会一直占用资源，也得不到释放，造成垂悬问题。答案是调用 `Destroy` 函数：

```c#
[NativeMethod(Name = "Scripting::DestroyObjectFromScripting", IsFreeFunction = true, ThrowsException = true)]
[MethodImpl(MethodImplOptions.InternalCall)]
public static extern void Destroy(Object obj, [DefaultValue("0.0F")] float t);
```



