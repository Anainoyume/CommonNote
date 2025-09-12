#### Unity 中 GUI, GUILayout, GUILayoutUtility, EditorGUI, EditorGUILayout, EditorGUIUtility 的区别

**1. 解释一下 Unity 中的编辑器扩展类，分为两组: **

- 在编辑器或者Runtime都能使用，命名空间UnityEngine
  - GUI： ui系统，包括button，lable，toggle等控件
  - GUILayout： 在GUI基础上，控件新增自动布局功能
  - GUILayoutUtility： 对布局类的一些补充，工具类。

- 只能在编辑器使用，命名空间UnityEditor
  - EditorGUI： 编辑器ui系统，和GUI非常相似，包括button，lable，toggle等控件
  - EditorGUILayout： 在EditorGUI基础上，控件新增自动布局功能
  - EditorGUIUtility： 对EditorGUILayout的一些补充，工具类。
    