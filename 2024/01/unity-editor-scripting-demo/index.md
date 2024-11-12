# Unity Editor Scripting 样例


Unity 的官方文档非常详细，但是不是所有人学习Unity都是从头看到尾看文档过来的。
我最近刚接触Unity,最近想给编辑器扩展一些功能，比如点击Inspector面板，在Project窗口中高亮选择Prefab或者Script。
这么小小的一个需求，查遍了中文互联网，不得其果。大家要么在CSDN上贴一些零碎的、没有格式化的代码，要么互相复制粘贴。结果不言而喻。
看外网，似乎大家提到了一些代码，但是Unity论坛上的人都是高手，作为一个新手看到了代码都不知道放在哪，怎么用，非常苦恼。
今天我把自己折腾了几个小时的一点小代码贴上，作为记录。如果大家对于编辑器扩展有大量需求，建议还是系统地学习一下官方的教程，不要像我，只求快，结果花费的时间更多。

1. 扩展代码需要放到Editor文件夹下
   这个Editor文件夹你可以建到任意的地方，我放到了Assets/Scripts/Editor下面，在其中创建一个C#文件，命名随意。
   ![20240104_unity_editor_location.png](/images/other/20240104_unity_editor_location.png)
2. 将以下代码粘贴进文件，保存，即可在Inspect面板点击挂载脚本时高亮脚本。

   ```c#
   using UnityEditor;
   using UnityEditor.UIElements;
   using UnityEngine;
   using UnityEngine.UIElements;
   
   
   [CustomEditor(typeof(MonoBehaviour), true)]
   public class HighlightItem : Editor {
       public override VisualElement CreateInspectorGUI() {
           var root = new VisualElement();
           InspectorElement.FillDefaultInspector(root, serializedObject, this);
           root.Add(new Label(&#34;点击以高亮&#34;));
           root.RegisterCallback&lt;ClickEvent&gt;(Highlight);
       
           return root;
       }
       
       private void Highlight(ClickEvent evt) {
           var script = MonoScript.FromMonoBehaviour((MonoBehaviour)target);
       
           UnityEditor.EditorGUIUtility.PingObject(script);
           //UnityEditor.Selection.activeObject = script;
       }
   }
   ```
当然这个功能Unity自带了，这个例子只是用来给像我这样完全没有接触过Unity的同学演示制作一个简单的扩展脚本的步骤。
Youtube上有很多更详细直观的教程，建议有条件有需要的同学看一遍。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/01/unity-editor-scripting-demo/  

