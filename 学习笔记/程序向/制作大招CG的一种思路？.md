## 制作大招镜头的一种思路

> 来源于 `晴天呼呼` 的 `绝区零战斗系统 demo`

从按下 大招按键 那一刻开始，**InputSystem** 受到信号, 执行 **OnFinishSkill** 事件，这个事件干两件事：

1. 输入 连招配置文件, 存在一个总的玩家连招配置文件 `PlayerComboSOData`，里面存放不同的 `ComboData`，按照种类区分存在普通攻击，派生攻击，大招，连携技等等。统一配置好，由系统将数据输入。接着会执行当前 Skill，播放音效特效动画。

<img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317131550487.png" alt="image-20250317131550487" style="zoom:33%;" />                    <img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317131616795.png" alt="image-20250317131616795" style="zoom:33%;" />   

2. 切换 `ComboStateMachine` 的状态为 `SkillState`，这个状态概括来说只做了一件事，那便是调控 相机切换器 `CameraSwither` 切换到 `FinishSkill State-Driven Camera` 状态轨道相机。

现在来详细说一下相机切换器，`CameraSwither` 是一个继承自 Monobehaviour 的单例类，它被挂载在主相机上：

<img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317131500652.png" alt="image-20250317131500652" style="zoom: 50%;" />

状态轨道相机组件如图所示，拖入 `Animator` 组件，在下方指定对应的状态，则当进入不同状态时，会切换到不同的子虚拟相机，由于     `Default Blend` 设置为 **剪切**。因此不同子虚拟相机切换时不会有过渡。

<img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317131800946.png" alt="image-20250317131800946" style="zoom: 50%;" />

`Anbi大招1` 是普通的虚拟相机，会先对着 Anbi 的脸拍，此时会执行一个准备放大招的动作。

而 `Anbi大招2` 和 `Anbi大招3` 都是轨道相机，需要结合 `CinemachineSmoothPath` 预先指定轨道的轨迹，之后拖入相机的预设中，这些轨迹被放置在角色的子物体之下：

<img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317132207027.png" alt="image-20250317132207027" style="zoom:33%;" />                 <img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317132221126.png" alt="image-20250317132221126" style="zoom:33%;" />

注意将 `Body` 的类型设置为 `Tracked Dolly`，拖入路径即可。

同时会注意到在过渡中存在 `OnCameraLive` 的一个事件，该事件在虚拟相机进入 `Live` 状态时会执行一次。而注意到这里拖入了存在 `Timeline` 的空物体。调用了 `TimelineItem.PlayTimeline` 函数，这个是大佬自己写的函数，内容仅仅是播放 `Timeline`。

注意到在播放大招镜头2时，有一段时间的慢放，我们来研究 `Timeline`：

<img src="C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250317132708710.png" alt="image-20250317132708710" style="zoom:50%;" />

注意到存在两个信号接收器，该作用类似于帧动画事件，而该帧事件通过调整 `Time.timeScale` 进行时间缩放。

而所有镜头动画播放完之后，通过动画事件，切换回主相机。