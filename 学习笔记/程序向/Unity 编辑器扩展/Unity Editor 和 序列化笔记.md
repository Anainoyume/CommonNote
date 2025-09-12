## Unity Editor 和 序列化那些事

先讲序列化，首先提一下 `SerializableAttribute` 这个特性，这个特性是 C# 自带的，原本是为了配合旧时代的 C# 二进制序列化而存在，大抵可以理解为一种规范。如果带有这个特性标签表示该类可以被序列化，但是现在的需多序列化模型都利用反射直接获取数据，这个特性用的比较少了。

> 这里是笔者后期插入的内容：我们顺带聊一聊什么是序列化和反序列化，把对象转换为字节序列的过程称为对象的序列化，而把字节序列恢复为对象的过程称为对象的反序列化。举一个最简单的例子，网络需要发送数据，我们可以将对象先序列化为 Json 数据，待另一头接收到了，由另一头反序列化为程序中的对象数据。

不过 Unity 仍然使用这个特性对自定义类的序列化进行标记，标记了就代表这个类可以序列化。而 Unity 有一套自己的序列化模型，并且提供了 `SerializedObject` 和 `SerializedProperty` 用来对序列化对象进行方便的操作。

尽管 Unity 的序列化器能序列化大部分的对象，但就好比楼观剑斩不断的东西几乎不存在一样。Unity 序列化器像类似 字典 这种常见的数据结构也是无法序列化的，而为了处理这样的情况。Unity 提供了两个供手动处理它们的回调函数，以便 Unity 能够序列化和反序列化这些数据类型。

下面是 `ISerializationCallbackReceiver` 接口需要实现的两个函数。

下面是一个字典序列化的例子，说是序列化其实非常的简陋，就是用两个 `List` 模拟了字典的键值对。然后在这两个关键时间节点存放而已。

```c#
// 在序列化之前会进行的操作
public void OnBeforeSerialize()
{
    _keys.Clear();
    _values.Clear();        
    foreach (var kvp in _myDictionary) {
        _keys.Add(kvp.Key);
        _values.Add(kvp.Value);
    }
}    

// 在反序列化之后会进行的操作
public void OnAfterDeserialize()
{
    _myDictionary = new Dictionary<int, string>();        
    for (int i = 0; i != Math.Min(_keys.Count, _values.Count); i++)
        _myDictionary.Add(_keys[i], _values[i]);
}
```



然后是 Unity 里关于序列化干嘛用的，我们可以定位到 `UnityEngine.Object` 有一个方法是用来复制实例的。像 C# 这种面向对象语言，引用对象默认赋值行为就是复制一份引用过去。那么面临拷贝的时候，就会涉及到深拷贝，浅拷贝的问题。一种方法就是创建一个新对象，然后逐个复制每一个字段的值。但试想，如果引用对象里又有引用对象的字段。这一过程就会非常不好。Unity 则采用了先将对象进行序列化，再将序列化之后的数据反序列化为 C# 对象。

而 Unity 序列化的数据采用的是 `Yaml` 标记语言，这是一种专门用来进行序列化标记的语言，和 `Json` 类似。而对于 Unity 底层的序列化实现，则是采用 C++ 实现的。具体代码逻辑被封装进了 dll 动态库内，Unity 在应用层只提供调用。

而 Unity 序列化还有一个就是有关于 Inspector Editor，Unity 会将符合他们规范的数据自动进行序列化处理，然后在 Inspector 上进行显示。大致逻辑便是 Unity 将 C# 对象以及数据 序列化为 序列化对象，再绘制上 Inspector，而在 Inspector Editor 的操作，则通过 `SerializedObject` 和 `SerializedProperty` 写回序列化对象。最后由 Unity 将其反序列化回 C# 对象。

Unity 的序列化还涉及到一些更深层次的例如 引用对象 的序列化，以及处理循环引用的问题，还有在热重载上面的应用。但这些过于深入的知识，笔者暂且未深入了解，故暂且不做记录。

---

聊完了序列化，接下来来说一下 Unity 关于自定义编辑器，自定义窗口，自定义属性绘制和 UI系统 的那些事。

先从 UI 系统开始说吧。Unity 的旧时代，基本采用的是 IMGUI 进行绘制，什么是 IMGUI 呢？转为化英文全称即：`Immediate Mode GUI`。也就是 `即时模式GUI` ，它会在每一帧都重新绘制所有的 UI，因此它即消耗效率，也不会保存持久性数据，依托每一帧的状态切换来决定自己应该画什么。举一个最简单的例子，例如滑动条控件，你需要为他提供一个 `Vector2` 的参数，告诉他滑动到什么位置了。而且每一帧都需要将这个数据进行写回，否则你的滑动条就根本动不了。但 IMGUI 也有一些优点，那就是写起来毕竟简单，实现一些快速的开发很直观，不过一遇到复杂的逻辑，就有点 hold 不住了。

因此 Unity 在现代推出了 UI Toolkit，核心是采用视觉树，还有会进行数据的持久保存。也就是只有需要重绘的控件才绘制，不需要所有的都在每一帧全部重画。视觉树使得 UI布局 简洁明了，结构清晰。而数据持久，大大节省了 UI 的性能消耗。但是据笔者搜集资料了解，尽管这个 UI Toolkit，官网在有关 UI 的词条中，都以它进行举例，并且也是 Unity 在未来推崇的 UI 系统，但是尚有很多功能不完善。因此使用的用户并不是很多，大部分包括 Odin Inspector 在内的插件，仍然使用 IMGUI 系统进行开发。国内则更倾向采用 `ugui`, `ngui` 等 UI 架构，但这两种架构笔者尚未了解。先不写。

那么接下来我们以 Unity 自带的 IMGUI 为例，讲述一下怎么编写自定义编辑器。

首先是聊一下上述提到的自定义编辑器，自定义窗口，自定义属性绘制的关系。其实 Unity 还有一个什么装饰器绘制，那个笔者没有去了解。因此这里也先不说了。首先自定义窗口可以单独提出来，它用于绘制独立的 Unity 窗口，例如像 Unity 里 Animator 采用的 动画器 那样的窗口。在现在可以简单使用 UIElement ( 貌似也是 UI Toolkit 的一部分 ) 和 Unity 提供的一个工具，来像 Qt 那样拖拽控件绘制 UI。

你可以通过这样的方式制作自己的工具窗口，当然你也可以选择使用代码。你需要在你的窗口类中继承 `EditorWindow` 类，这就可以了。为了打开你的窗口，你需要在你的 `CustomWindow` 类中提供一个方法，创建一个 `CustomWindow` 的实例，并将其显示，一个例子看上去是下面这样：
``` C#
public class MyWindow : EditorWindow
{

    // 这个特性使得在 Unity 编辑器中会提供一个菜单栏，供你调用下面的方法
    [MenuItem("Window/CustomWindow/My Window")]
    private static void Init()
    {
        // 使用 GetWindow 方法获取 window，如果已经存在就返回。没有则创建一个新的 window 实例。
        // 是 Unity 自带的方法
        MyWindow window = (MyWindow)GetWindow(typeof(MyWindow));
        // 可写可不写, GetWindow 之后会自带显示你的窗口
        // window.Show();
    }

    // 启动窗口时调用的初始化
    private void OnEnable() {

    }

    // IMGUI 的每一帧重绘
    private void OnGUI()
    {

    }
}
```

通过这种方式，你就可以创建你自己的自定义窗口了。

接下来说一下：自定义编辑器和自定义属性绘制，属性绘制是在编辑器的内部进行调用的。编辑器绘制是指你在 Inspector 看到的所有界面，全部由 Editor 进行全局绘制，而自定义属性绘制，则是细节上，它相当于提供了一个绘制接口，Editor 在某些绘制方法只负责调用这个接口。具体接口实现有属性绘制自己实现，也就是说你告诉 Editor 它要怎么样画自己，而 Editor 只负责画。因自定义此属性绘制是一个细节上的调控。

先来说自定义编辑器，它大概像下面这样：

```C#
using System;
using UnityEditor;
using UnityEngine;

// 告诉 Unity 哪些类需要重新绘制整个 Inspector 视图
[CustomEditor(typeof(MonoBehaviour), true)]
public class InspectorMonoEditor : Editor	// 必须继承 Editor，且整个类必须位于 Editor 文件夹下
{
    // 每次启动绘制时的初始化, 例如你切换到其他界面又切回这个界面，此时算作需要重新启动绘制
    private void OnEnable() {

    }

	// IMGUI 的绘制函数
    public override void OnInspectorGUI() {
        // 调用上层 OnInspectorGUI, 就是 Unity 的原生绘制
        // base.OnInspectorGUI();
    }

}      
```



