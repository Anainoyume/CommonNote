## Unity 自定义控件

这里先聊一件事就是 `UxmlFactory` 是什么。

我们实现一个自定义控件的时候通常会写如下代码：

```C#
public class InspectorView : VisualElement
{
    // 这是个啥？
    public new class UxmlFactory : UxmlFactory<InspectorView, UxmlTraits> {}

    public InspectorView() {

    }
}
```

从结果上来，如果不写这个。那么 UI Builder 里就不会出现我们的自定义控件，然后官方对这个方法也写的极为模糊。但是可以大致理解为每一个控件都需要依靠这个来进行实例化。对于我在网上搜集的资料：

> 嵌套的 `UxmlFactory` 类是 Unity 用来实例化自定义控件的方法，当在 Uxml 文件中遇到时。它允许您以声明方式定义UI，并让 Unity 根据这些定义自动处理 UI 元素的创建和设置。

因此每次实现的时候固定加上这一句就行了！